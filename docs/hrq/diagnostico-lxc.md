# Diagnóstico do LXC 120 para o fork do Habblaud

Preciso de um diagnóstico SOMENTE LEITURA deste ambiente para planejar uma
alteração no Habblaud. Não instale, altere, reinicie nem apague nada.
Não mostre tokens, chaves, senhas nem o conteúdo de auth.json/credentials.

1. Sistema: whoami, hostname, distro, versão do node, docker (se houver).
2. Claude Code: `claude --version`; quais pastas de config existem
   (~/.claude*, CLAUDE_CONFIG_DIR); liste só os nomes de primeiro nível
   e confirme se há `projects/` e `sessions/`.
3. Cursor: versão do CLI (`cursor-agent --version` ou equivalente);
   estrutura de ~/.cursor (só nomes, 2 níveis); existe hooks.json?
   Mostre o conteúdo dele se existir.
4. Hooks/plugins do Claude Code: em settings.json de cada conta, mostre
   só as seções "hooks" e "enabledPlugins" (nada mais).
5. Habblaud: como está rodando (docker ps / processo node / systemd),
   de qual pasta e versão; valores de HABBLAUD_HOST, HABBLAUD_BIND,
   HABBLAUD_ALLOWED_HOSTS, HABBLAUD_TERMINAL e HABBLAUD_CLAUDE_DIRS;
   resultado de `curl -s http://127.0.0.1:4747/api/health`.
   Descubra também em qual pasta ele foi instalado: se for Docker, veja o
   label com.docker.compose.project.working_dir do container; se for Node,
   o cwd do processo; em último caso, procure um package.json com
   "name": "habblaud" (ignore node_modules).
6. Acesso externo: o Habblaud é acessado por Nginx Proxy Manager,
   Tailscale ou IP direto? Qual porta está publicada (ss -ltnp)?
7. Os agentes da fábrica rodam como qual usuário e a partir de quais
   pastas de projeto?

No fim, me dê um resumo em tópicos, curto, pronto para colar em outra
conversa. Grave o mesmo resumo em ~/diagnostico-habblaud.txt.
