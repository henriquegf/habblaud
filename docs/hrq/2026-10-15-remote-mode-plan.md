# Modo remoto nativo — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Com `HABBLAUD_PASSWORD`, o Habblaud aceita aprovar/mensagens/terminal pelo navegador em host não local (ex. Tailscale), atrás de sessão com cookie; sem senha, comportamento idêntico ao original.

**Architecture:** Módulo `server/auth/` guarda sessões em memória e compara a senha com SHA-256 + `timingSafeEqual`. `isPrivileged(req)` = `Host` loopback **ou** cookie válido. Com senha, `terminal`/`messages` ligam mesmo com bind na rede. Rotas sensíveis e o snapshot/stream exigem privilégio; hooks do Codex/Claude continuam só em loopback. O cliente consulta `/api/auth/session` antes do EventSource e mostra login quando necessário.

**Tech Stack:** Node 22+, TypeScript, Vitest, HTTP nativo (sem Express), cliente vanilla TS (Vite).

**Spec:** `docs/hrq/2026-10-15-remote-mode-design.md`  
**Branch de implementação:** `feat/remote-mode` a partir de `main` (código candidato a PR no original). Docs em `docs/hrq/` podem ficar na `hrq` ou ser cherry-pickados; o PR upstream não precisa de `docs/hrq/`.

## Global Constraints

- Sem `HABBLAUD_PASSWORD` (ausente, vazia ou só espaço) = comportamento do Habblaud original.
- Cookie: nome `habblaud_session`, valor 32 bytes hex, `HttpOnly`, `SameSite=Strict`, `Path=/`, `Max-Age=604800` (7 dias); `Secure` só em HTTPS / `X-Forwarded-Proto: https`.
- Sessão só na memória do processo (restart = desloga).
- `POST /api/codex/events` nunca aceita sessão remota.
- CSRF: só o guard atual (JSON + Origin) + `SameSite=Strict`.
- Commits: Conventional Commits em inglês (`feat(auth): ...`).
- Verificação a cada tarefa relevante: `npm test` e, no fim, `npm run typecheck`.
- Fora de escopo: Cursor adapter, OAuth, token por máquina, HTTPS obrigatório.

## File map

| File | Responsibility |
| --- | --- |
| `server/config.ts` | `authPassword` / flag; `terminal`/`messages` ligados quando há senha |
| `server/auth/password.ts` | Hash SHA-256 + comparação timing-safe |
| `server/auth/sessions.ts` | Map token→expiry; create/get/revoke |
| `server/auth/rate-limit.ts` | 5 falhas / IP → 60s |
| `server/auth/cookies.ts` | Parse `Cookie`, `Set-Cookie` / clear |
| `server/auth/service.ts` | `AuthService`: required, privileged, login/logout/session |
| `server/auth/*.test.ts` | Testes unitários do módulo |
| `server/http/guard.ts` | (opcional) reexport ou nada — privilégio fica no auth |
| `server/http/app.ts` | Rotas auth; gate 401; privilegiado nas rotas fechadas |
| `server/http/sessions.ts` | `sessionsLockError` aceita privilegiado |
| `server/http/sse.ts` | `local` = loopback **ou** autenticado |
| `server/index.ts` | Instancia `AuthService`, passa ao handler e ao Hub |
| `server/http/auth.test.ts` | Testes de integração HTTP da auth e das travas |
| `client/src/net/auth.ts` | `fetchSession`, `login`, `logout` |
| `client/src/ui/login.ts` | Tela de senha |
| `client/src/main.ts` | Gate de boot antes de `store.connect()` |
| `client/src/ui/permission.ts` (+ usos) | Privilegiado = local **ou** autenticado |
| `client/src/ui/settings.ts` | Botão Sair quando auth required |
| `client/src/net/store.ts` | Emitir/reagir a 401 no stream |
| `server/README.md`, `README.md` | Documentar `HABBLAUD_PASSWORD` |

---

### Task 1: Config — senha liga terminal/messages na rede

**Files:**
- Modify: `server/config.ts`
- Modify: `server/http/terminal.test.ts` (bloco `trava do terminal (config)`)
- Test: mesmo arquivo / ou `server/config` via `terminalOffReason` já testado em `terminal.test.ts`

**Interfaces:**
- Produces: `export function authPassword(env): string | undefined` — trim; `undefined` se vazio
- Produces: `ServerConfig` ganha `authPassword?: string` (valor bruto só em memória; nunca logar)
- Produces: `terminalOffReason` / `messagesOffReason` — se `authPassword(env)` definido, ignoram a trava de bind (ainda respeitam `HABBLAUD_TERMINAL=0` / `HABBLAUD_MENSAGENS=0`)

- [ ] **Step 1: Write the failing test**

Em `server/http/terminal.test.ts`, dentro de `describe('trava do terminal (config)')`:

```ts
it('com HABBLAUD_PASSWORD: liga mesmo com porta exposta', () => {
  expect(terminalOffReason({ HABBLAUD_PASSWORD: 'segredo' }, '0.0.0.0', false)).toBeUndefined();
  expect(terminalOffReason({ HABBLAUD_PASSWORD: 'segredo', HABBLAUD_BIND: '0.0.0.0' }, '0.0.0.0', true)).toBeUndefined();
  expect(terminalOffReason({ HABBLAUD_PASSWORD: '  ' }, '0.0.0.0', false)).toBeDefined();
  expect(terminalOffReason({ HABBLAUD_PASSWORD: 'segredo', HABBLAUD_TERMINAL: '0' }, '0.0.0.0', false)).toMatch(/HABBLAUD_TERMINAL/);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run server/http/terminal.test.ts -t "com HABBLAUD_PASSWORD"`  
Expected: FAIL (ainda desliga com porta exposta)

- [ ] **Step 3: Implement**

Em `server/config.ts`:

```ts
/** Senha do modo remoto (trim); undefined = desligado. */
export function authPassword(env: NodeJS.ProcessEnv = process.env): string | undefined {
  const p = env.HABBLAUD_PASSWORD?.trim();
  return p || undefined;
}
```

No início de `terminalOffReason`, depois do check de `HABBLAUD_TERMINAL`:

```ts
if (authPassword(env)) return undefined;
```

Em `loadConfig`, incluir `authPassword: authPassword(env)` no retorno (adicionar campo em `ServerConfig`).

- [ ] **Step 4: Run tests**

Run: `npx vitest run server/http/terminal.test.ts`  
Expected: PASS (incluindo o novo e os antigos sem senha)

- [ ] **Step 5: Commit**

```bash
git add server/config.ts server/http/terminal.test.ts
git commit -m "$(cat <<'EOF'
feat(auth): enable terminal when HABBLAUD_PASSWORD is set

EOF
)"
```

---

### Task 2: Módulo `server/auth` — senha, sessão, cookie, rate limit

**Files:**
- Create: `server/auth/password.ts`
- Create: `server/auth/sessions.ts`
- Create: `server/auth/rate-limit.ts`
- Create: `server/auth/cookies.ts`
- Create: `server/auth/service.ts`
- Create: `server/auth/password.test.ts`, `sessions.test.ts`, `rate-limit.test.ts`, `cookies.test.ts`, `service.test.ts`

**Interfaces:**
- Produces:
  - `hashPassword(password: string): Buffer` — SHA-256
  - `passwordsEqual(a: string, b: string): boolean` — hash + `timingSafeEqual`
  - `SessionStore` — `create(): string`, `has(token: string): boolean`, `revoke(token: string): void`, TTL 7d
  - `LoginRateLimit` — `tooMany(ip: string): boolean`, `fail(ip: string): void`, `success(ip: string): void` (5 / 60s)
  - `parseCookie(header, name): string | undefined`
  - `sessionSetCookie(token, { secure, maxAge }): string`
  - `sessionClearCookie({ secure }): string`
  - `createAuthService(opts: { password: string; now?: () => number }): AuthService`
  - `AuthService`:
    ```ts
    {
      readonly required: true;
      isAuthenticated(req: IncomingMessage): boolean;
      isPrivileged(req: IncomingMessage): boolean; // isLoopbackHost(host) || isAuthenticated
      login(req, res, password: string): { status: 200 } | { status: 401 | 429; error: string };
      logout(req, res): void;
      session(req): { required: true; authenticated: boolean };
      /** Para Secure no Set-Cookie */
      cookieSecure(req: IncomingMessage): boolean;
    }
    ```

- [ ] **Step 1: Write failing unit tests** (um arquivo por unidade; exemplos mínimos)

`password.test.ts`: iguais → true; diferentes → false; strings vazias não quebram.  
`sessions.test.ts`: create → has; revoke → !has; após TTL com `now` injetado → !has.  
`rate-limit.test.ts`: 5 fails → tooMany; success zera; após 60s com `now` injetado libera.  
`cookies.test.ts`: parse com vários cookies; Set-Cookie contém HttpOnly e SameSite=Strict; clear com Max-Age=0.  
`service.test.ts`: privileged com Host localhost sem cookie; não privileged com Host 10.0.0.1; login certo → authenticated; login errado 401; 5 errs → 429.

- [ ] **Step 2: Run tests — expect FAIL**

Run: `npx vitest run server/auth`

- [ ] **Step 3: Implement files**

Detalhes de cookie (spec):

```
habblaud_session=<64 hex>; HttpOnly; SameSite=Strict; Path=/; Max-Age=604800
```

(+ `; Secure` se `req.socket` encrypted ou `x-forwarded-proto === 'https'`).

Token: `randomBytes(32).toString('hex')`.

IP para rate limit: `req.socket.remoteAddress` (strip `::ffff:`).

- [ ] **Step 4: Run tests — expect PASS**

Run: `npx vitest run server/auth`

- [ ] **Step 5: Commit**

```bash
git add server/auth
git commit -m "$(cat <<'EOF'
feat(auth): add in-memory session and password helpers

EOF
)"
```

---

### Task 3: Rotas HTTP — auth + gate privilegiado

**Files:**
- Modify: `server/http/app.ts`
- Modify: `server/http/sessions.ts` (`sessionsLockError`)
- Modify: `server/http/sse.ts` (`Hub.attach` / `local`)
- Modify: `server/index.ts`
- Modify: `server/test/permission-server.ts` (e fixtures de teste que montam `createApiHandler`, se precisarem do novo dep)
- Create: `server/http/auth.test.ts`

**Interfaces:**
- Consumes: `AuthService` da Task 2; `authPassword` da Task 1
- Produces: `ApiDeps.auth?: AuthService` (ausente = sem modo remoto)
- Produces: helper interno `privileged(req) => !auth || auth.isPrivileged(req)`
- Produces: rotas `/api/auth/login|logout|session` só se `auth` presente; senão 404
- Produces: sem privilégio nas rotas fechadas → `401` `{ error: 'autenticação necessária' }` (não 403)
- Produces: `Hub` recebe `isLocalClient?: (req) => boolean` (default: `isLoopbackHost`); com auth: `(req) => auth.isPrivileged(req)`
- Produces: `/api/health` inclui `authRequired: boolean`
- **Não muda** a condição de `/api/codex/events` (continua loopback host + remoteAddress)

Rotas fechadas (spec): snapshot, stream, agents, sessions, terminal, permissions, messages (+ inbox/bridge), mod/summary, character, rooms/rename, stats, demo, timeline, updates.

Mensagens de 403 por “recurso desligado” (`!deps.terminal` etc.) permanecem 403 — só o “não é local” vira “não privilegiado” → 401 quando auth ligada; quando auth desligada, manter 403 e texto atual (`isLoopbackHost`).

Padrão sugerido por rota sensível:

```ts
const allow = deps.auth ? deps.auth.isPrivileged(req) : isLoopbackHost(req.headers.host);
if (!allow) {
  sendJson(res, deps.auth ? 401 : 403, {
    error: deps.auth
      ? 'autenticação necessária'
      : '…mensagem original de Host local…',
  });
  return true;
}
```

Para snapshot/stream (hoje abertos): se `deps.auth && !deps.auth.isPrivileged(req)` → 401.

- [ ] **Step 1: Write `server/http/auth.test.ts`** (servidor de teste com senha, `Host: 192.168.0.10` ou `habblaud.lan`)

Casos mínimos:
1. Sem `auth` no handler: `GET /api/auth/session` → 404; snapshot com Host local → 200.
2. Com auth, Host remoto sem cookie: snapshot/stream → 401; health → 200 + `authRequired: true`; session → `{ required: true, authenticated: false }`.
3. Login senha certa com Host remoto → 200 + `set-cookie`; snapshot com Cookie → 200.
4. Login errada → 401; 5ª → 429.
5. Logout → snapshot remoto 401 de novo.
6. Host `localhost` sem cookie + auth ligada → snapshot 200 e permissions não 401 por host.
7. `POST /api/codex/events` com cookie válido mas Host remoto → 403 (não libera).

Reusar o padrão de `request()` em `permission-server.ts` (Host trocável).

- [ ] **Step 2: Run — expect FAIL**

Run: `npx vitest run server/http/auth.test.ts`

- [ ] **Step 3: Wire `app.ts`, `sessions.ts`, `sse.ts`, `index.ts`**

Em `index.ts`:

```ts
import { createAuthService } from './auth/service';
const auth = config.authPassword ? createAuthService({ password: config.authPassword }) : undefined;
// Hub:
const hub = new Hub(office, { isLocalClient: auth ? (req) => auth.isPrivileged(req) : undefined });
// createApiHandler({ ..., auth })
```

Ajustar `sessionsLockError(enabled, host, privileged?: boolean)` ou passar `allow: boolean` para não importar auth em sessions.

- [ ] **Step 4: Run full related tests**

Run: `npx vitest run server/http/auth.test.ts server/http/http.test.ts server/http/terminal.test.ts server/permissions/http.test.ts server/messages/http.test.ts server/http/sessions.test.ts server/codex/http.test.ts`  
Expected: PASS (atualizar asserts antigos só se algum fixture quebrar por assinatura de `Hub`/`ApiDeps`)

- [ ] **Step 5: Commit**

```bash
git add server/http server/index.ts server/test/permission-server.ts
git commit -m "$(cat <<'EOF'
feat(auth): gate remote API behind session cookie

EOF
)"
```

---

### Task 4: Cliente — login, privilegiado, logout, 401

**Files:**
- Create: `client/src/net/auth.ts`
- Create: `client/src/net/auth.test.ts`
- Create: `client/src/ui/login.ts`
- Modify: `client/src/main.ts`
- Modify: `client/src/ui/permission.ts` — exportar `isPrivilegedHostname` ou aceitar flag `authenticated`
- Modify: `client/src/ui/composer.ts`, `drawer.ts`, `character-editor.ts`, `roomrename.ts` — usar privilegiado
- Modify: `client/src/ui/settings.ts` — botão Sair
- Modify: `client/src/ui/context.ts` / store — estado `authenticated` / `authRequired`
- Modify: `client/src/net/store.ts` — em erro HTTP 401 do EventSource, emitir evento `auth:required` (ou callback)
- Modify: `client/src/ui/styles.css` — estilos mínimos da tela de login (seguir tokens existentes; sem redesign do escritório)
- Test: `client/src/ui/permission.test.ts`, `client/src/net/auth.test.ts`

**Interfaces:**
- Produces:
  ```ts
  // net/auth.ts
  export type AuthSession = { required: boolean; authenticated: boolean };
  export async function fetchAuthSession(signal?: AbortSignal): Promise<AuthSession>; // 404 → { required: false, authenticated: false }
  export async function login(password: string): Promise<'ok' | 'unauthorized' | 'rate_limited' | 'error'>;
  export async function logout(): Promise<void>;
  ```
- Produces: `isClientPrivileged(hostname, authenticated: boolean): boolean` = `isLocalHostname(hostname) || authenticated`
- Produces: `main.ts` não chama `store.connect()` até auth ok; se precisa login, monta `LoginScreen` e só então conecta

Fluxo boot:

```ts
const session = await fetchAuthSession();
if (session.required && !session.authenticated) {
  await showLogin(); // resolve após login ok
}
store.setAuth({ required: session.required, authenticated: true }); // ou equivalente
store.connect();
```

401 no meio: `store` em `error` do EventSource — se possível ler status (EventSource não expõe bem); alternativa prática: ao falhar reconnect, `GET /api/auth/session` e se `required && !authenticated` → `disconnect` + callback para `main` mostrar login de novo. Também: fetches de permission/message que retornam 401 disparam o mesmo.

- [ ] **Step 1: Testes unitários**

- `fetchAuthSession`: mock fetch 404 → not required; 200 body → parse.  
- `isClientPrivileged('100.x.x.x', true) === true`; `(..., false) === false`; localhost false auth → true.  
- Login screen pode ter teste leve de DOM se o projeto já testa UI assim; senão cobrir só net/permission.

- [ ] **Step 2: Run — expect FAIL**

Run: `npx vitest run client/src/net/auth.test.ts client/src/ui/permission.test.ts`

- [ ] **Step 3: Implement cliente**

Textos PT-BR alinhados ao app: “Senha”, “Entrar”, “Senha incorreta”, “Tente de novo em um minuto”, “Sair”.

- [ ] **Step 4: Run client + typecheck**

Run: `npx vitest run client/src/net/auth.test.ts client/src/ui/permission.test.ts`  
Run: `npm run typecheck`  
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add client
git commit -m "$(cat <<'EOF'
feat(auth): add browser login for remote mode

EOF
)"
```

---

### Task 5: Docs + verificação final

**Files:**
- Modify: `server/README.md` (tabela de variáveis + parágrafo da trava do terminal)
- Modify: `README.md` (tabela de variáveis + menção ao modo remoto / senha)
- Modify: `docs/hrq/CONTEXTO.md` — passo 2 como implementado / pendente só validação no Tailscale

- [ ] **Step 1: Documentar `HABBLAUD_PASSWORD`**

Texto sugerido (PT, como o resto do README):

> `HABBLAUD_PASSWORD` — se definida, liga o modo remoto: terminal/permissões/mensagens funcionam com a porta na rede, e o navegador em host não local precisa da senha (cookie de sessão). Sem ela, o comportamento continua o de sempre (terminal só com bind local + Host local).

Atualizar frases que dizem “não há como ligar com a porta exposta” para excepcionar a senha.

- [ ] **Step 2: Suite completa**

Run: `npm test && npm run typecheck`  
Expected: tudo verde

- [ ] **Step 3: Smoke manual no LXC (checklist)**

1. Subir fork em `:4748` com `HABBLAUD_PASSWORD=...` e host/bind na rede (não substituir `:4747`).
2. No LXC: `http://127.0.0.1:4748` → entra sem senha; aprovar/terminal ok.
3. No Windows/celular via Tailscale: pede senha; após login, aprovar + mensagem + terminal.
4. Sem variável de senha: porta exposta → terminal desligado como no original.

- [ ] **Step 4: Commit**

```bash
git add server/README.md README.md docs/hrq/CONTEXTO.md
git commit -m "$(cat <<'EOF'
docs(auth): document HABBLAUD_PASSWORD remote mode

EOF
)"
```

---

## Self-review (plan vs spec)

| Spec requirement | Task |
| --- | --- |
| Senha `HABBLAUD_PASSWORD` | 1, 2 |
| Cookie HttpOnly / SameSite / Secure condicional | 2 |
| Sessão em memória, 7 dias, logout | 2, 3 |
| Rate limit 5 / 60s | 2, 3 |
| Local OU autenticado | 3, 4 |
| Escritório remoto fechado (snapshot/stream 401) | 3 |
| codex/events sem sessão remota | 3 |
| SSE localSize com autenticado | 3 |
| Cliente login / logout / 401 | 4 |
| Sem senha = original | 1, 3, 5 |
| Docs + critério Tailscale | 5 |
| Cursor adapter | **fora** (passo 3 / `feat/cursor-adapter`) |

## Nota: Cursor

Este plano **não** lê agentes do Cursor. Isso é o passo 3 do `CONTEXTO.md` (`feat/cursor-adapter`: `~/.cursor/chats/` + hooks do `cursor-agent`, espelhando `server/sources/codex/`). O modo remoto só desbloqueia o navegador para o que o servidor já observa (hoje Claude Code e, se ligado, Codex).
