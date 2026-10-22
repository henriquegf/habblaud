# Modo remoto nativo — design

Branch: `feat/remote-mode` (nasce de `main`).  
Contexto: `docs/hrq/CONTEXTO.md` (passo 2).  
Caminho B: autenticação no navegador, sem gateway que contorne a trava.

## Objetivo

Abrir o Habblaud no LXC pelo Tailscale (ou outro host na rede) e usar aprovar, mensagens e terminal sem túnel SSH, com senha no navegador. Sem senha configurada, o comportamento permanece o do original.

## Decisões

| Tema | Escolha |
| --- | --- |
| Credencial | Senha em `HABBLAUD_PASSWORD` |
| Sessão | Cookie de sessão opaco, estado só na memória do processo |
| Escritório remoto | Fechado sem sessão (`/api/snapshot`, `/api/stream` e o resto sensível → 401) |
| Localhost | Continua privilegiado sem login |
| Hooks (Claude/Codex) | Continuam só em `127.0.0.1`; sessão remota não os autentica |
| CSRF | O que já existe: `Content-Type: application/json` + `Origin` + cookie `SameSite=Strict` |

## Arquitetura da trava

### Sem `HABBLAUD_PASSWORD`

Nada muda. `terminalOffReason` / `messagesOffReason` seguem desligando o recurso se a porta estiver exposta. As rotas sensíveis exigem `Host` local (`isLoopbackHost`).

### Com senha

A senha substitui a trava de “porta só no loopback”:

- `terminal` e `messages` ligam mesmo com `HABBLAUD_HOST` / `HABBLAUD_BIND` na rede.
- `HABBLAUD_TERMINAL=0` e `HABBLAUD_MENSAGENS=0` continuam desligando o recurso.
- Uma requisição é privilegiada se o `Host` for loopback **ou** o cookie de sessão for válido.

| Quem chama | O que acontece |
| --- | --- |
| `localhost` / `127.0.0.1` | Escritório, aprovar, mensagens, terminal, personagem e renomear sala, sem login. Plugins e hooks seguem em `127.0.0.1`. |
| Outro host, com sessão | O mesmo. |
| Outro host, sem sessão | `401` nas rotas fechadas (abaixo). |
| Qualquer um | Página, assets, `GET /api/health`, `GET /api/auth/session`, `POST /api/auth/login`, `POST /api/auth/logout`. |

Rotas fechadas sem privilégio (host remoto sem sessão): snapshot, stream, agentes, sessões, terminal, permissões, mensagens (incluindo inbox/bridge do mod e do Codex), `GET /api/mod/summary`, personagem, renomear salas, stats, demo, timeline e updates.

`POST /api/codex/events` **não** aceita sessão remota. Continua só com `Host` local e, fora do Docker, conexão vinda do loopback.

O hub SSE (`server/http/sse.ts`) conta como “página local” quem está no loopback **ou** autenticado, para o escritório saber que dá para responder permissão (`localSize`).

## Sessão e autenticação

### Senha

- `HABBLAUD_PASSWORD` com algum caractere (após trim) liga o modo.
- Vazio ou só espaço = desligado.
- Comparação: SHA-256 dos dois lados + `timingSafeEqual`.
- A senha não entra em log.

### Cookie `habblaud_session`

- Valor aleatório de 32 bytes em hex (64 caracteres), guardado só na memória do processo.
- Flags: `HttpOnly`, `SameSite=Strict`, `Path=/`, `Max-Age` = 7 dias.
- `Secure` só quando a conexão é HTTPS ou `X-Forwarded-Proto` é `https` (Tailscale em HTTP ainda grava o cookie).
- Reiniciar o processo ou passar dos 7 dias exige login de novo.
- Logout apaga a entrada na memória e o cookie.

### Rotas de auth

Só existem quando a senha está ligada. Sem senha, respondem `404`.

| Rota | Comportamento |
| --- | --- |
| `POST /api/auth/login` `{ password }` | `200` + `Set-Cookie`, ou `401` |
| `POST /api/auth/logout` | `200`; apaga sessão e cookie; vale mesmo sem sessão |
| `GET /api/auth/session` | `{ required: boolean, authenticated: boolean }` |

`GET /api/health` ganha `authRequired: true` quando a senha está ligada e continua aberto sem autenticação.

### Limite de tentativas

Cinco senhas erradas pelo mesmo IP → `429` por 60 segundos. Um acerto zera a contagem. No Docker o IP visto pode ser o gateway (trava compartilhada); aceitável para um operador só.

## Cliente

1. Antes do EventSource, `GET /api/auth/session`. `404` (senha desligada) ou falha de rede trata-se como `{ required: false, authenticated: false }` — escritório sobe como hoje.
2. Se `required === false` ou `authenticated === true` → escritório como hoje.
3. Se `required && !authenticated` → tela só com senha e “Entrar” (página/assets ok; snapshot/stream só depois do login).
4. Login: `POST /api/auth/login`. `200` abre o escritório; `401` “senha incorreta”; `429` “tente de novo em um minuto”.
5. Privilegiado no UI: `isLocalHostname` **ou** autenticado (aprovar, mensagens, terminal, personagem, renomear). O servidor é a barreira real.
6. “Sair” no menu (só com `authRequired`): `POST /api/auth/logout`, fecha o stream, volta à tela de senha.
7. Qualquer `401` em snapshot/stream/ação no meio do uso → fecha o stream e volta à tela de senha, sem reload completo.

## Implementação (mapa no código)

| Área | Arquivos prováveis |
| --- | --- |
| Config | `server/config.ts` — ler senha; ajustar `terminal`/`messages` quando auth ligada |
| Auth | `server/auth/` (novo) — hash, sessões em memória, rate limit, parse/set cookie |
| Guard / privilégio | `server/http/guard.ts` ou helper em `auth` — `isPrivileged(req)` = loopback host **ou** sessão válida |
| Rotas | `server/http/app.ts` — auth routes; trocar `isLoopbackHost` por privilegiado nas rotas fechadas; manter regra estrita em `/api/codex/events` |
| SSE | `server/http/sse.ts` — `local` inclui autenticado |
| Health | `server/http/app.ts` — campo `authRequired` |
| Cliente | tela de login; sessão no boot; `isLocalHostname` → privilegiado; logout; tratamento de 401 |
| Docs | `server/README.md` / README — documentar `HABBLAUD_PASSWORD` |

## Deploy no LXC

- Branch `feat/remote-mode` a partir de `main`.
- Fork sobe em outra porta (ex. `4748`) com `HABBLAUD_PASSWORD` e host/bind na rede ou Tailscale.
- Original em `127.0.0.1:4747` permanece intocado.
- Sem senha, o fork se comporta como o original.

## Fora deste PR

- Token por máquina / multi-operador
- HTTPS obrigatório / TLS terminado no Habblaud
- OAuth
- Gateway, shipper ou SMB
- Adapter do Cursor (`feat/cursor-adapter`)
- Segundo mecanismo de CSRF além do guard atual

## Critério de pronto

1. De outra máquina, abrir pelo IP do Tailscale, digitar a senha e conseguir aprovar, mandar mensagem e ver o terminal — sem túnel SSH.
2. Em `localhost` no LXC, tudo segue sem login.
3. Sem `HABBLAUD_PASSWORD`, comportamento idêntico ao original (incluindo terminal desligado com porta exposta).
4. `npm test` e `npm run typecheck` verdes.

## Testes (mínimo)

**Servidor**

- Senha ausente → comportamento atual (terminal desligado com bind na rede; Host remoto → 403 nas rotas sensíveis).
- Senha presente → terminal/messages ligam com bind na rede.
- Host local sem cookie → privilegiado.
- Host remoto sem cookie → 401 nas rotas fechadas; 200 em health, session e login.
- Cookie válido → libera rotas fechadas.
- Logout e restart do processo → invalidam sessão.
- Senha errada → 401; quinta falha → 429.
- `POST /api/codex/events` continua sem aceitar sessão remota.

**Cliente**

- Tela de login quando `required && !authenticated`.
- Controles liberados com sessão (mesmo hostname não local).
- 401 no meio do uso volta ao login.
