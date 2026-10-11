# claude-config

Configuração pessoal do Claude Code: este repo **é** a pasta `~/.claude`.

| Item | O que é |
|---|---|
| `skills/<nome>/` | 13 skills do pacote "nativas" do Claude (criação de skills e agentes, brainstorming, MCP, humanizer etc.) |
| `settings.json` | configurações, sem segredo |

Fora do git, de propósito: login, histórico, conversas, plugins e `CLAUDE.md` pessoal.

## Licenças

Cada skill mantém a licença de origem, quando existe um arquivo `LICENSE`:

| Skill | Licença |
|---|---|
| `artifacts-builder`, `mcp-builder`, `skill-creator` | Apache 2.0 |
| `humanizer`, `prompt-improver` | MIT |
| demais (`ai-agent-workspace`, `brainstorming`, `chrome-devtools-cli`, `create-skill`, `create-subagent`, `dispatching-parallel-agents`, `subagent-driven-development`, `writing-skills`) | sem arquivo de licença no pacote; origem não verificada |

As skills da Anthropic com "All rights reserved" **não** estão aqui.
