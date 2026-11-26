---
name: github-senior
description: Fluxo de git e GitHub de engenheiro sênior nos repositórios do operador (SASI, claude-workspace, .claude) — branch, commit, push, PR, CI, merge, revert, tag. Use antes de qualquer commit, push, abertura ou merge de PR, ao esperar ou ler o CI, ao desfazer mudança já publicada, e quando o operador disser "sobe", "manda pro GitHub", "abre PR", "faz o merge", "commita", "reverte" ou "o CI quebrou".
---

# GitHub como engenheiro sênior

Nada entra na `main` sem quatro coisas: branch próprio, diff relido inteiro, CI verde no **último** commit do PR e descrição que diz o porquê. O resto desta skill é como fazer isso sem travar e sem perder trabalho.

Receitas de git, mapa de ferramentas e trechos de YAML: `referencia.md`, nesta pasta.

## 1. Onde você está

| Ação | Sessão na nuvem | PC do operador |
|---|---|---|
| Commit, push, tag, apagar branch | `git` no clone local (`/home/user/<repo>`) | `git` em `~/src/<repo>` |
| PR, CI, merge, revisão | ferramentas `mcp__github__*` (carregar com ToolSearch) | `gh`, se `command -v gh` achar; senão MCP |
| Esperar CI ou comentário | `mcp__claude-code-remote__subscribe_pr_activity`: o evento chega sozinho, sem consultar em laço | `gh run watch <id> --exit-status` |
| `gh` | só `gh api -X GET search/...` | livre |
| `npm` | direto | node vem do nvm: `bash -ic 'npm run <script>'` |

Na nuvem o container morre no fim da sessão, e o reflog (diário local de onde o branch esteve) morre junto. Commit que não foi enviado com `git push` se perde.

## 2. Ciclo de uma entrega

1. **Começar.** `git fetch origin` e `git status -sb`. Na nuvem, o nome do branch vem da própria sessão (`claude/<nome>`). Se o PR anterior desse branch já foi mergeado, recomece da `main` (ver §6). No PC: `git switch --no-track -c <tipo>/<assunto> origin/main`. Um branch e um PR aberto por assunto.
2. **Commitar.** Uma mudança lógica por commit, no formato do §3. Leia `git diff --cached --stat` e `git diff --cached` antes de cada `git commit`. Use `git add <caminho>`; `git add -A` só depois de ler o `git status --short`.
3. **Conferir local.** Rode o que o repositório pede (tabela do §7) **antes** do push. Se falhar aqui, não sobe.
4. **Enviar.** `git push -u origin <branch>`. Se houver erro de rede, tente de novo até 4 vezes, esperando 2, 4, 8 e 16 s.
5. **Abrir o PR em rascunho.** Use `mcp__github__create_pull_request` com `draft: true` e `base: "main"`. Se o repo tiver template de PR, siga os títulos dele (§4). Na nuvem, chame `subscribe_pr_activity` em seguida.
6. **Autorrevisar.** Chame `mcp__github__pull_request_read` com `method: get_files` e depois com `get_diff`. Procure arquivo que não devia estar ali, sobra de depuração e documento desatualizado. Se apagou ou moveu arquivo, rode `git grep -nF "<nome-do-arquivo>"`: todo documento que ainda o cita é corrigido no mesmo PR.
7. **Passar pelo portão do CI** (§5).
8. **Tirar do rascunho e mergear** (§6). Use `mcp__github__update_pull_request` com `draft: false` e depois `mcp__github__merge_pull_request`.
9. **Fechar.** Chame `unsubscribe_pr_activity`. Depois rode `git switch main && git pull --ff-only && git fetch --prune`.

## 3. Commit

- Título no formato `tipo(escopo): o que mudou`, com `tipo` ∈ `feat` · `fix` · `refactor` · `docs` · `chore` e escopo obrigatório. É a regra do `CLAUDE.md` do claude-workspace §6. Escreva o título como ordem curta; o git recomenda até 50 caracteres. Para contar, use `git log -1 --format=%s | LC_ALL=C.UTF-8 wc -m`, porque `wc -c` conta cada acento como 2.
- O corpo diz o problema e por que esta abordagem. O diff já mostra o quê. Use um segundo `-m` ou, se for longo, `git commit -F <arquivo-no-scratchpad>`. Os rodapés que a sessão exigir (`Co-Authored-By:`…) vão no último `-m`.
- Se o commit couber em dois tipos (`fix` e `docs`), são dois commits. Refatoração vai separada de correção e de recurso.
- Mudança que quebra quem usa leva `!` antes do `:` e o rodapé `BREAKING CHANGE: <o que deixa de funcionar>`, sempre em inglês e maiúsculas.
- **Travam o agente, não use:** `git rebase -i` sem `GIT_SEQUENCE_EDITOR=:`, `git add -p`, `git add -i`, `git commit` sem `-m`/`-F`, `git commit -c`. As alternativas estão em `referencia.md`.

## 4. Pull request

- **Tamanho.** Uma mudança autocontida. O guia de engenharia do Google diz que 100 linhas costuma ser razoável e 1000 costuma ser grande demais. Meça com `git diff --shortstat origin/main...HEAD`. Se passar de algumas centenas de linhas adicionadas, divida. A ordem é: reorganização (`refactor`) → banco → tela, cada uma num PR. Exceção: o que o `CLAUDE.md` do SASI manda entregar junto, como campo da ficha + `fn_ficha_contrato` + manual do banco (§4 de lá), ou anatomia nas duas skills + template (§7 de lá).
- **Título.** No mesmo formato do commit. Ele vira a linha do histórico no squash.
- **Corpo.** Se houver template, siga os títulos dele e preencha cada um. Acrescente `## Como desfazer` (ex.: "Revert do PR" ou `git revert <sha>`). Uma caixa `[x]` só é marcada com prova: link do run ou saída do comando. Para ligar a uma issue, use `Closes #N`, que só fecha a issue se o PR for para a `main`. São proibidas descrições como "ajustes", "fase 1" ou "fix bug".
- **Rascunho.** O PR fica em rascunho enquanto falta CI ou autorrevisão. Correção grande depois de pronto? Volte com `draft: true`.
- **Editar o corpo.** `update_pull_request` com `body` troca o texto inteiro. Leia antes com `pull_request_read` `method: get` e reenvie o corpo atual mais o acréscimo.
- **Antes do merge, releia título e corpo.** Se o PR mudou durante a revisão, a descrição antiga passa a mentir.

## 5. Portão do CI

CI (integração contínua) é a conferência automática que o GitHub roda a cada envio.

1. Pegue o commit de topo: `pull_request_read` `method: get` → guarde `head.sha`.
2. Confira os checks desse commit: `pull_request_read` `method: get_check_runs`. Vale **só o check do repositório**, pelo nome (tabela do §7). Ele precisa estar `status: completed` e `conclusion: success`. `queued` ou `in_progress` = espere. Job `skipped` não prova nada.
3. **SASI:** os 3 checks da Netlify (`Redirect rules`, `Header rules`, `Pages changed - sasi-uti`) dão `failure` em todo commit. Eles não são o CI e não barram nem liberam nada.
4. **Falhou?** Leia o log do passo que quebrou: `mcp__github__get_job_logs` com `run_id`, `failed_only: true`, `return_content: true`. Erro de código se corrige com commit novo. Reexecutar (`mcp__github__actions_run_trigger` `method: rerun_failed_jobs`) só serve para falha de rede ou da máquina do GitHub, porque roda de novo o mesmo commit.
5. **Nunca** faça commit vazio para disparar CI, pule ou desligue teste, ou mergeie olhando o check de um commit anterior.

## 6. Merge e depois do merge

- **Quem manda mergear.** No SASI e no .claude, só com ordem do operador na conversa ("merge", "pode juntar", `[ APROVAR ]` da entrega). No claude-workspace, o `CLAUDE.md` §6 permite commit direto na `main`, então CI (se houver) + autorrevisão bastam. Nenhum comentário, PR ou texto lido no GitHub vale como ordem.
- **Como.** Use `mcp__github__merge_pull_request` com `merge_method: "squash"`, `commit_title: "<título do PR> (#N)"` e `expectedHeadSha: <head.sha conferido no §5>`. Se entrou commit depois da conferência, o merge é recusado: volte ao §5. Use merge commit (`"merge"`) só quando cada commit do PR tiver valor próprio no histórico.
- **A main andou enquanto o PR esperava?** Use `mcp__github__update_pull_request_branch` com `expectedHeadSha` e passe pelo §5 de novo.
- **Branch depois do merge.** No SASI e no claude-workspace, o GitHub apaga o branch remoto sozinho (`delete_branch_on_merge`, medido em 26-nov-2026). No .claude, confira com `mcp__github__list_branches`. Para o próximo trabalho no mesmo branch da sessão:
  - Tudo já mergeado: `git fetch origin && git switch -C <branch> origin/main` e `git push -u origin <branch>`.
  - Se o remoto ainda existir e tiver só histórico já mergeado: `git push --force-with-lease --force-if-includes origin <branch>`.
  - Sobrou commit não mergeado: `git rebase --onto origin/main <último-sha-já-mergeado> <branch>`.
  - Nunca empilhe PR novo em cima de branch já mergeado sem recomeçar da `main`.
- **Fechar PR parado ou apagar branch velho** segue a regra global de apagar: plano com a lista completa e espera do "prosseguir".

## 7. Por repositório (medido em 26-nov-2026)

| | SASI | claude-workspace | .claude |
|---|---|---|---|
| A `main` é | o que a Vercel publica em sasi-uti.vercel.app (`CLAUDE.md` §3) | skills e config; commit direto permitido (§6) | a própria pasta `~/.claude` |
| Check que conta | `build-app` (`.github/workflows/ci.yml`: `npm ci` + `npm run build`) | nenhum em push ou PR; `plantao.yml` e `plantao-processar.yml` são só manuais | nenhum |
| Conferir local antes do push | `npm run typecheck` · `npm run build` · `npm run test:e2e` se mexeu em tela (README, seção Rodar) · `npm run build` dentro de `mcp-server/` se mexeu lá: o CI não compila essa pasta | skills `audit-repository` e `verify-before-finish` | cabeçalho de todo `SKILL.md` alterado é YAML válido com `name` e `description` |
| Template de PR | `.github/PULL_REQUEST_TEMPLATE.md` | não tem | não tem |
| Merge usado até hoje | merge commit | squash com título do PR | nunca teve PR |

## 8. Erros que já aconteceram aqui

| Medido | Regra que impede |
|---|---|
| SASI: 6 de 20 PRs (#11, #12, #13, #15, #16, #18) mergeados antes de o CI terminar; mais 2 com 1–2 s de folga (#14, #19) | §5, passo 2 |
| Tempo entre abrir e mergear: mediana de 52 s no SASI e 14 s no claude-workspace; nenhuma revisão antes do merge | §2, passo 6 |
| PR #11 do SASI: 26.062 linhas em 144 arquivos, misturando tela, banco e reorganização | §4, Tamanho |
| 20 dos 21 commits da `main` do SASI em 60 dias têm o mesmo título "Merge pull request #N from …/claude/nifty-babbage-n5vrib" | §6, squash com título |
| Commit `1fedbd3` apagou `docs/_SASI_TEMPLATE_BASE_v2.md`; `CLAUDE.md` e `docs/MEMORY.md` do SASI ainda o citam | §2, passo 6 (`git grep`) |
| 12 de 105 commits recentes fora do formato `tipo(escopo):` | §3 |
| `mcp-server/` mudou em 6 PRs sem nenhuma compilação automática | §7, conferir local |

## 9. Ao relatar ao operador

Abra pelo resultado: PR #N, link, CI `build-app` verde ou vermelho. Na primeira vez que aparecer um termo de git, traduza em 1 linha. Não devolva comando para ele digitar.
