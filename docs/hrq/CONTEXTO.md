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
| `hrq` | este contexto + junção das features para rodar no LXC |

## Próximos passos

1. Teste sem código: túnel `ssh -L 4747:127.0.0.1:4747 henrique@100.124.226.33` e abrir
   `http://localhost:4747` no Windows. Confirmar que aprovar/mensagens/terminal funcionam.
2. `feat/remote-mode`: desenhar a autenticação do navegador (sessão com senha ou token,
   cookie `HttpOnly`/`Secure`, CSRF) e trocar a checagem de `Host` local por
   "local OU autenticado" nas rotas de terminal, permissões, mensagens e personagem.
   Manter o padrão atual (sem autenticação configurada = comportamento original).
3. `feat/cursor-adapter`: investigar o formato de `~/.cursor/chats/` e os hooks do
   `cursor-agent`; criar `server/sources/cursor/` seguindo o Codex.
4. Rodar `npm test` e `npm run typecheck` a cada etapa.
