# Referência — git e GitHub sem travar

Leia só a seção que a tarefa pede. Os comandos foram testados no git 2.43.0 do container da nuvem (26-nov-2026), salvo onde estiver marcado. Apagar ou sobrescrever qualquer coisa (stash, worktree, branch, PR) segue a regra global de apagar: plano com a lista completa e espera do "prosseguir".

## 1. Receitas de git

| Preciso | Comando |
|---|---|
| Ver os commits locais que ainda não foram enviados | `git log --oneline @{u}..HEAD`; sem upstream: `git log --oneline HEAD --not --remotes` |
| Refazer o último commit local | `git reset --soft HEAD^` → ajustar → `git add <arquivos>` → `git commit -C ORIG_HEAD` (mesma mensagem) ou `-m "<nova>"` |
| Separar um commit local que misturou dois assuntos | `git reset HEAD~1` → `git add <arquivos do fix>` → `git commit -m "fix(...): ..."` → `git add <docs>` → `git commit -m "docs(...): ..."` |
| Corrigir um commit local mais antigo (só se ainda não foi enviado) | `git commit --fixup=<sha>` → `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash --autostash <sha>~1`. Sem o `-i`, o autosquash não funde nesta versão (medido) |
| Preparar só parte de um arquivo | `git diff <arquivo> > <scratchpad>/p.patch` → apagar os blocos `@@` que não entram → `git apply --cached --check <scratchpad>/p.patch` → repetir sem `--check`. Se as duas mudanças caem no mesmo bloco: gere com `-U0` e aplique com `--unidiff-zero` |
| Desfazer commit já publicado | `git revert --no-commit <sha>` (merge commit: `git revert --no-commit -m 1 <sha>`) → `git commit -m "fix(<escopo>): desfaz <assunto>" -m "This reverts commit <sha>."`. Sempre num branch novo, por PR. O `--no-edit` gera o título `Revert "..."`, fora do formato |
| Trazer a `main` para o branch | branch só local: `git rebase origin/main` · branch já enviado: `git merge origin/main` · na `main`: `git pull --ff-only` |
| Terminar um conflito | `git status` lista os arquivos. Resolvido: `git add <arquivos>`, depois `git commit --no-edit` (merge) ou `GIT_EDITOR=true git rebase --continue` (rebase). Sem saber resolver: `git merge --abort` ou `git rebase --abort`. Conflito de `git stash apply` ou `pop`: `git reset --merge`; o stash continua em `git stash list` |
| Achar trabalho "perdido" | `git reflog -n 20` → `git branch resgate HEAD@{N}`. Commit fora de qualquer branch sai do reflog em 30 dias; o resto, em 90 |
| Achar o commit que quebrou | `git bisect start <sha-ruim> <sha-bom>` → `git bisect run npm run typecheck` (no PC: `git bisect run bash -ic 'npm run typecheck'`) → anote o "first bad commit" → `git bisect reset`. Saída 125 = pular o commit |
| Guardar trabalho pela metade | `git stash push -u -m "<motivo>"` → `git stash list` → `git stash apply stash@{0}` → `git status`. O stash fica guardado até o `drop`, que é apagar |
| Urgência no meio de outra mudança | `git worktree add -b fix/<assunto> ../<repo>-urgente origin/main` → trabalhar lá → `git push -u origin fix/<assunto>`. A pasta nova não tem `node_modules`. No PC (8 GB de RAM), um `npm run dev` por vez. `git worktree remove` apaga a pasta e os ignorados dela (`node_modules`) sem avisar |
| Marcar versão | `git tag -a v0.1.0 -m "<resumo>"` → `git push origin v0.1.0`. Tag enviada nunca se move: se errou, crie `v0.1.1` |

**Versão (SemVer: MAJOR.MINOR.PATCH).**
- `fix` sobe o PATCH, `feat` sobe o MINOR e `BREAKING CHANGE` sobe o MAJOR.
- Na fase `0.y.z`, a quebra sobe o número do meio (`0.4.1` → `0.5.0`).
- Ir para `1.0.0` é decidir que o sistema está estável. A primeira versão é `0.1.0`.

## 2. Ferramentas MCP de GitHub

Na nuvem há dois servidores: `mcp__github__` (minúsculo, o completo) e `mcp__Github__` (maiúsculo). Use o minúsculo. O maiúsculo serve só para as buscas `search_*`. Em sessão nova, confirme os nomes com ToolSearch antes de usar.

| Preciso | Ferramenta e parâmetros |
|---|---|
| Ler o template de PR | `get_file_contents` `path: .github/PULL_REQUEST_TEMPLATE.md` |
| Abrir PR | `create_pull_request` `title`, `head`, `base: main`, `body`, `draft: true` |
| PRs de um branch | `list_pull_requests` `head: "doutortenente:<branch>"`, `state: "all"` |
| Dados do PR e `head.sha` | `pull_request_read` `method: get` |
| Arquivos, diff, commits | `pull_request_read` `method: get_files` · `get_diff` · `get_commits` |
| Checks do commit de topo | `pull_request_read` `method: get_check_runs` (jobs do Actions e da Netlify; o `id` de cada um é o `job_id`) · `get_status` (status de commit, onde aparecem a Vercel e a Netlify) |
| Runs de um workflow | `actions_list` `method: list_workflow_runs`, `resource_id: ci.yml`, `workflow_runs_filter: {branch, event}` · jobs: `method: list_workflow_jobs` |
| Log do que falhou | `get_job_logs` `job_id`, `return_content: true` · ou, para o run inteiro: `run_id`, `failed_only: true`, `tail_lines: 200` |
| Texto de um check que chegou por evento | `get_check_run` `checkRunId` |
| Reexecutar, disparar, cancelar | `actions_run_trigger` `method: rerun_failed_jobs` · `rerun_workflow_run` · `run_workflow` (só em workflow com `workflow_dispatch`; exige `workflow_id` e `ref`) · `cancel_workflow_run` |
| Tirar do rascunho, editar, fechar | `update_pull_request` `draft: false` · `title` · `body` (substitui o corpo inteiro) · `state: closed` (fechar é apagar: lista + "prosseguir") |
| Trazer a `main` para o PR | `update_pull_request_branch` `expectedHeadSha` |
| Mergear | `merge_pull_request` `merge_method`, `commit_title`, `commit_message`, `expectedHeadSha` |
| Revisão com comentário em linha | `pull_request_review_write` `method: create` → `add_comment_to_pending_review` → `pull_request_review_write` `method: submit_pending`, `event: COMMENT` |
| Responder e fechar comentário de revisão | `pull_request_read` `method: get_review_comments` → `add_reply_to_pull_request_comment` `commentId`, `pullNumber`, `body` → `resolve_review_thread` `threadId` (`PRRT_...`) |
| Comentário na conversa do PR | `add_issue_comment` `issue_number: <nº do PR>` |
| Branches | `list_branches` · criar: `create_branch` `branch`, `from_branch: main` · apagar: não há ferramenta; depois da lista + "prosseguir", `git push origin --delete <branch>` |
| Issue | ler: `issue_read` · criar ou fechar: `issue_write` (ao fechar, `state_reason`) · listar: `list_issues` (`state: OPEN`/`CLOSED`, em maiúsculas). O MCP não busca issue por texto |
| Histórico | `list_commits` `sha: <branch>`, `path` · `get_commit` · `search_commits` (maiúsculo) só procura na branch padrão |
| Tag e release | só leitura: `list_tags`, `get_tag`, `list_releases`, `get_latest_release`. A tag se cria pelo git local |
| Gravar arquivo sem clone | `push_files` (vários arquivos num commit). Com clone na mão, prefira o git local |
| Vigiar o PR | `mcp__claude-code-remote__subscribe_pr_activity` / `unsubscribe_pr_activity` (`owner`, `repo`, `pullNumber`) |

Nenhuma ferramenta MCP lê ou muda as opções do repositório, como o método de merge.

## 3. No PC, com `gh`

Não foi conferido se o `gh` está instalado no PC: rode `command -v gh` antes. Os comandos abaixo foram lidos no manual do gh:

- **Abrir em rascunho:** `gh pr create --draft --base main --title "..." --body-file <arquivo>`. Tirar do rascunho: `gh pr ready <n>`.
- **Esperar o CI do SASI:** `gh run list --workflow ci.yml --commit "$(git rev-parse HEAD)" --limit 1 --json databaseId --jq '.[0].databaseId'` → `gh run watch <id> --exit-status`.
  - Saída vazia quer dizer que o run ainda não nasceu: rode de novo.
  - Não use `gh pr checks --fail-fast` no SASI: os checks da Netlify falham sempre e encerram a espera.
- **Log do que falhou:** `gh run view --job <JOB_ID> --log-failed`. Reexecutar só o que falhou: `gh run rerun <RUN_ID> --failed`.
- **Reproduzir a falha do CI:** o CI usa Node 20 e compila o PR já somado à `main`.
  1. `git fetch origin pull/<N>/merge && git worktree add ../<repo>-ci FETCH_HEAD`
  2. Dentro da pasta nova: `bash -ic 'nvm install 20 && nvm exec 20 npm ci && nvm exec 20 npm run build'`.
- **Release com notas geradas a partir dos PRs:** `gh release create v0.1.0 --verify-tag --generate-notes`.

## 4. CI — trechos de YAML para quando mexer num workflow

Mudar workflow do SASI é configuração de produção: proponha no PR e espere `[ APROVAR ]`.

```yaml
on:
  push:
    branches: [ main ]
  pull_request:          # sem paths: — com filtro, o check pode ficar pendente para sempre
  workflow_dispatch:     # rodar à mão; esse run NÃO aparece nos checks do PR

concurrency:             # cancela o run velho do mesmo PR quando chega commit novo
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  build-app:             # não renomear: é o nome conferido no §5 da skill
    runs-on: ubuntu-latest
    timeout-minutes: 10  # o padrão é 360 min; o CI do SASI levou de 32 a 57 s
```

- Para guardar foto ou relatório de teste, use `actions/upload-artifact` com `if: ${{ !cancelled() }}`. Para baixar: `actions_list` `method: list_workflow_run_artifacts`, depois `actions_get` `method: download_workflow_run_artifact`.
- Para reconferir um PR, reexecute o run do evento `pull_request`. Run de `workflow_dispatch` não conta como check do PR.
