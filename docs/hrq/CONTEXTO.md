# Fork HRQ do Habblaud — contexto para continuar no LXC 120

Leia este arquivo antes de mexer no código. Ele resume o que já foi decidido.
Plano completo: https://claude.ai/code/artifact/84334374-9b70-4077-a228-9ca7c422c1db

## Objetivo

Um Habblaud no LXC 120 (`fabrica`) que mostra e controla os agentes do Claude Code
e do Cursor, acessível de outras máquinas (navegador, celular) sem perder aprovar,
mensagens e terminal.

## Ambiente (diagnóstico de 12/out/2026)

- LXC `fabrica`: Ubuntu 24.04, Node 22, Docker 29, Tailscale (100.124.226.33).
- Claude Code 2.1.295, uma conta em `~/.claude`; agentes rodam como `henrique` em `~/projetos`.
- Habblaud 0.8.0 ORIGINAL rodando em Docker a partir de `/home/henrique/habblaud`,
  porta só em `127.0.0.1:4747`. Plugins `habblaud`, `habblaud-permissoes` e
  `habblaud-mensagens` ativos, com marketplace apontando para `/home/henrique/habblaud`.
- Cursor: `cursor-agent` 2026.10.01 (CLI), conversas em `~/.cursor/chats/`, sem `hooks.json`.
- Este fork NÃO deve ser clonado por cima de `/home/henrique/habblaud` enquanto o
  original estiver em uso.

## O que o código mostrou (Fase 1)

- Leitura de sessões é polling de 1 s (`server/sources/watcher.ts`); `fs.watch` só acelera.
- Aprovar, mensagens e terminal só ligam com bind local E `Host` local
  (`server/config.ts` → `terminalOffReason`; `server/http/guard.ts`; rotas em
  `server/permissions/http.ts`, `server/messages/http.ts`). Hooks falam só com `127.0.0.1`.
- O Codex é o molde para o Cursor: fonte em `server/sources/codex/` (interface
  `AgentSource` em `server/sources/source.ts`) + rota `POST /api/codex/events`
  (`server/codex/http.ts`) + hook `mod/habblaud-codex/hook.mjs`.

## Decisões

- Caminho B: modo remoto NATIVO (autenticação por token/sessão no navegador em vez de
  exigir `Host` local), auditável, candidato a PR para o original. Nada de gateway que
  contorne a trava.
- Como tudo roda no mesmo LXC, NÃO há shipper nem SMB. O modo remoto cobre só o
  acesso pelo navegador de outra máquina. Token por máquina fica para quando entrar
  uma segunda máquina com agentes.
- Aprovação remota sempre atrás de autenticação forte; nunca aberta na internet.

## Branches

| Branch | Uso |
| --- | --- |
| `main` | espelho do original (marmottajr/habblaud); sem commits próprios |
| `feat/remote-mode` | modo remoto (PR 2 para o original) — nasce de `main` |
| `feat/cursor-adapter` | fonte do Cursor (PR 1 para o original) — nasce de `main` |
| `feat/idle-leave` | saída por inatividade `HABBLAUD_IDLE_LEAVE_MIN` (PR para o original) — nasce de `main` |
| `hrq` | este contexto + junção das features para rodar no LXC |

`feat/cursor-adapter` depende de `feat/idle-leave` (o Cursor usa a mesma variável como janela de presença):
os PRs para o original saem nesta ordem: `feat/idle-leave`, depois `feat/cursor-adapter`. `feat/remote-mode`
é independente. Correção numa feature: commit na branch dela e merge para a frente
(`feat/idle-leave` → `feat/cursor-adapter` → `hrq`); nunca corrigir direto na `hrq`.

## Estado em 22/out/2026

- `hrq` tem as três features juntas; testes e typecheck verdes.
- Serviço `habblaud.service` (systemd do usuário, porta 4749, `npm run dev` na worktree `feat-cursor-adapter`)
  com `HABBLAUD_IDLE_LEAVE_MIN=10` no drop-in `~/.config/systemd/user/habblaud.service.d/idle-leave.conf`.
- Docker na 4747 segue o 0.8.0 original.

## Saída por inatividade (`feat/idle-leave`)

- Antes, o Claude Code só saía do escritório quando a sessão fechava (`sessions/<pid>.json` some); os 10 min valiam
  só para o Cursor. Agora `HABBLAUD_IDLE_LEAVE_MIN` (desligada por padrão) vale para todas as fontes, no `Office`.
- Principal ocioso há N min, sem subagente rodando e sem permissão pendente, sai (aparece `offline` por 20 s e some;
  sala vazia some). Volta ao trabalhar, pedir permissão/pergunta, esperar shell ou em `/clear`/`/resume`.
- No boot conta do último evento do transcript. No Cursor, a variável é a janela de presença (sem ela, 10 min).
- QA (22/out): liberado com ressalvas. Ressalva 1 (pedido de permissão de quem saiu) corrigida em `bd21c59`.
  Observações em aberto: agente fora segue acessível por id (intencional); no Cursor a saída usa o aviso de sessão
  encerrada, não o de inatividade.

## Próximos passos

1. Teste sem código: túnel `ssh -L 4747:127.0.0.1:4747 henrique@100.124.226.33` e abrir
   `http://localhost:4747` no Windows. Confirmar que aprovar/mensagens/terminal funcionam.
2. `feat/remote-mode`: implementado na worktree `.worktrees/feat-remote-mode`
   (branch `feat/remote-mode`, 9 commits à frente de `main`). Desenho/plano em
   `docs/hrq/2026-10-15-remote-mode-{design,plan}.md`. Pendente: smoke Tailscale
   no LXC e decisão de PR/merge. Cursor = passo 3.
3. `feat/cursor-adapter`: implementado na worktree `.worktrees/feat-cursor-adapter`
   (MVP observação via `meta.json` + uso do plano no topo: Cursor Models / Other Models
   via `GetCurrentPeriodUsage` e `~/.config/cursor/auth.json`). Desenho/plano em
   `docs/hrq/2026-10-15-cursor-adapter-{design,plan}.md` e
   `docs/hrq/2026-10-19-cursor-usage-{design,plan}.md`. Pendente: smoke no LXC e
   PR/merge. Fases 2–3 (terminal/hooks) depois.
4. `feat/idle-leave`: pronta; abrir PR para o original antes do PR do Cursor.
5. Rodar `npm test` e `npm run typecheck` a cada etapa.
