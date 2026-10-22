# Adapter Cursor (observação) — design

Branch: `feat/cursor-adapter` (nasce de `main`).  
Contexto: `docs/hrq/CONTEXTO.md` (passo 3).  
Molde: `server/sources/codex/` + `AgentSource` em `server/sources/source.ts`.

## Objetivo (MVP)

Mostrar no escritório os agentes do Cursor Agent / CLI que têm conversa recente em
`~/.cursor/chats/`, com sala pelo `cwd` e título da conversa. **Só observar:** sem
terminal, sem aprovar, sem mensagens pelo Habblaud.

O desenho deixa caminho claro até a paridade com o Codex (fases 2 e 3 abaixo).

## Achados do ambiente (LXC)

- Layout: `~/.cursor/chats/<projectHash>/<chatId>/meta.json` + `store.db` (SQLite).
- `meta.json`: `schemaVersion`, `createdAtMs`, `updatedAtMs`, `hasConversation`,
  `title`, `cwd`, `isSubagent` (opcional).
- `store.db`: tabelas `blobs` (JSON) e `meta` (hex → JSON com `agentId`,
  `blobEncryptionKey`, etc.). Blobs hoje legíveis em plaintext; o MVP **não** os lê.
- Sem `hooks.json` do Cursor no host; `cursor-agent` tem plugins, não o mesmo modelo
  de hooks do Claude/Codex.
- `Provider` atual: `'claude' | 'codex'` — o MVP acrescenta `'cursor'`.

## Decisões

| Tema | Escolha |
| --- | --- |
| Escopo MVP | Observação (presença + sala + título + status grosso) |
| Fonte de verdade | Só `meta.json` + mtime de `store.db` |
| Conta | Sintética `.cursor` apontando para `~/.cursor` |
| Desligar | `HABBLAUD_CURSOR=0` |
| Subagentes | Fora do MVP |
| Terminal / hooks | Fora do MVP; previstos na pasta |

## Arquitetura

### `CursorSource` (`server/sources/cursor/`)

Implementa `AgentSource` com `provider: 'cursor'`.

- **Boot:** `office.beginBoot` / `endBoot` em `finally`; varre chats e registra principais abertos.
- **Polling:** ~1 s (e `fs.watch` em `chats/` quando útil), no espírito do Codex/Claude.
- **Presença:** chat “aberto” se `updatedAtMs` **ou** mtime de `store.db` estiver dentro de
  `CURSOR_RECENT_MS` (padrão 30 min, constante exportada e testável).
- **Id:** `.cursor:<chatId>` (estável enquanto o diretório existir).
- **Sala:** `cwd` do `meta.json` (normalizado). Sem `cwd` utilizável → não entra no Office.
- **Título:** `title` do meta (fallback curto se ausente).
- **Status:** dentro da janela e “quente” (ex. atualizado nos últimos poucos minutos) →
  `working`; ainda na janela mas frio → `idle`; fora da janela → remove / `offline` como as
  outras fontes.
- **Atividade:** opcional genérica (“Em conversa”) quando `working` — sem parse de tools.
- **`transcriptPathOf`:** `undefined` no MVP (sem terminal).
- **`terminalParser`:** ausente no MVP.

Ligado em `server/index.ts` no `SourceSet` quando a config permitir e existir pasta de chats
(mesmo padrão do Codex: sem pastas → fonte não sobe).

### Conta e config

- Conta `.cursor` registrada via `AccountsService` / detecção espelhando o Codex em
  escala menor: pasta `~/.cursor` ou override `HABBLAUD_CURSOR_DIR` /
  `HABBLAUD_CURSOR_DIRS` (lista) para o futuro Docker.
- `ServerConfig`: flag `cursor` (ligado por padrão se não desligado; só sobe a fonte se
  houver `chats/`).
- Docs: `HABBLAUD_CURSOR` na tabela de variáveis (README / server/README).

### Protocolo e cliente

- `shared/types.ts`: `Provider = 'claude' | 'codex' | 'cursor'`.
- Cliente: `PROVIDER_NAME`, chip (letra sugerida `R`), textos mínimos. Sem UI de
  permissão/mensagem/terminal específica do Cursor no MVP (`canMessage` /
  `permission` ausentes).

## Roadmap (mesma pasta)

| Fase | Entrega |
| --- | --- |
| **1 (este PR)** | Presença + salas via `meta.json` (+ mtime) |
| **2** | Leitura de `store.db` → `transcriptPathOf` / parser / histórico |
| **3** | `CursorLive` + `POST /api/cursor/events` + plugin/hook → aprovar e mensagens |

Interfaces e ids do MVP devem permanecer válidos nas fases 2–3 (não inventar um segundo
sistema de identidade).

## Testes (mínimo)

- Fixture com árvore fake `chats/<hash>/<id>/meta.json` (+ `store.db` vazio ou touch).
- Chat recente com `cwd` → agente no Office na sala certa.
- Chat velho / sem `cwd` → não aparece.
- `HABBLAUD_CURSOR=0` → fonte não liga.
- `provider: 'cursor'` no snapshot / sources.
- `npm test` e `npm run typecheck`.

## Critério de pronto (MVP)

1. Com `cursor-agent` (ou IDE) ativo em um projeto no LXC, o Habblaud do fork mostra um
   agente Cursor na sala desse `cwd`, com título coerente.
2. Parar de usar / esfriar a conversa → o agente some após a janela.
3. Claude (e Codex, se ligado) continuam iguais.
4. Suite e typecheck verdes.

## Fora deste PR

- Parse de `store.db` / terminal / histórico
- Hooks, aprovar, mensagens
- Subagentes Cursor
- Montagem Docker de `~/.cursor`
- Merge na branch `hrq` com `feat/remote-mode` (integração operacional depois)
