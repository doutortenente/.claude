# Referência — git e GitHub sem travar

Ler só a seção que a tarefa pede. Os comandos foram testados no git 2.43.0 do container da nuvem (26-nov-2026), salvo onde estiver marcado.

## 1. Receitas de git

| Preciso | Comando |
|---|---|
| Ver só os commits locais ainda não enviados | `git log --oneline @{u}..HEAD` · sem upstream: `git log --oneline HEAD --not --remotes` |
| Refazer o último commit local | `git reset --soft HEAD^` → ajustar → `git add <arquivos>` → `git commit -C ORIG_HEAD` (mesma mensagem) ou `-m "<nova>"` |
| Separar um commit local que misturou dois assuntos | `git reset HEAD~1` → `git add <arquivos do fix>` → `git commit -m "fix(...): ..."` → `git add <docs>` → `git commit -m "docs(...): ..."` |
| Corrigir um commit local mais antigo | `git commit --fixup=<sha>` → `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash --autostash <sha>~1` (sem `-i`, o autosquash não funde nesta versão; medido) |
| Preparar só parte de um arquivo | `git diff <arquivo> > <scratchpad>/p.patch` → apagar os blocos `@@` que não entram → `git apply --cached --check <scratchpad>/p.patch` → o mesmo sem `--check`. Duas mudanças no mesmo bloco: gerar com `-U0` e aplicar com `--unidiff-zero` |
| Desfazer commit já publicado | `git revert --no-edit <sha>` (commit comum ou de squash) · `git revert -m 1 <sha>` (merge commit). Sempre num branch novo, por PR |
| Trazer a `main` para o branch | só local: `git rebase origin/main` · já enviado: `git merge origin/main` · na `main`: `git pull --ff-only` |
| Sobrescrever o próprio branch de PR depois de rebase | `git push --force-with-lease --force-if-includes origin <branch>`. Nunca na `main`. Nunca `--force` puro. Não tire o `--force-if-includes`: com o `git fetch` antes, o `--force-with-lease` sozinho perde a garantia |
| Achar trabalho "perdido" | `git reflog -n 20` → `git branch resgate HEAD@{N}`. Commit fora de qualquer branch some do reflog em 30 dias; o resto, em 90 |
| Achar o commit que quebrou | `git bisect start <sha-ruim> <sha-bom>` → `git bisect run npm run typecheck` (PC: `git bisect run bash -ic 'npm run typecheck'`) → anotar "first bad commit" → `git bisect reset`. Saída 125 = pular o commit |
| Guardar trabalho pela metade | `git stash push -u -m "<motivo>"` → `git stash list` → `git stash apply stash@{0}` → `git status` → `git stash drop stash@{0}` |
| Urgência no meio de outra mudança | `git worktree add -b fix/<assunto> ../<repo>-urgente origin/main` → trabalhar lá → `git push -u origin fix/<assunto>` → `git worktree remove ../<repo>-urgente`. A pasta nova não tem `node_modules`. No PC (8 GB de RAM), um `npm run dev` por vez |
| Conflito em merge, rebase ou stash | `git status` lista os arquivos. Não sabe resolver? `git merge --abort` / `git rebase --abort` e volte ao estado anterior |
| Marcar versão | `git tag -a v0.1.0 -m "<resumo>"` → `git push origin v0.1.0`. Tag enviada nunca se move: errou, sai `v0.1.1` |

**Versão (SemVer, MAJOR.MINOR.PATCH).** `fix` sobe o PATCH, `feat` sobe o MINOR e `BREAKING CHANGE` sobe o MAJOR. Na fase `0.y.z`, a quebra sobe o número do meio (`0.4.1` → `0.5.0`). Passar para `1.0.0` é a decisão de declarar o sistema estável. A primeira versão é `0.1.0`.

## 2. Ferramentas MCP de GitHub

Há dois servidores na nuvem: `mcp__github__` (minúsculo, o completo) e `mcp__Github__` (maiúsculo). Use o minúsculo. O maiúsculo serve só para as buscas `search_*`. Em sessão nova, confirme os nomes com ToolSearch antes de usar.

| Preciso | Ferramenta e parâmetros |
|---|---|
| Ler o template de PR | `get_file_contents` `path: .github/PULL_REQUEST_TEMPLATE.md` |
| Abrir PR | `create_pull_request` `title`, `head`, `base: main`, `body`, `draft: true` |
| Dados do PR e `head.sha` | `pull_request_read` `method: get` |
| Arquivos, diff, commits | `pull_request_read` `method: get_files` · `get_diff` · `get_commits` |
| Checks do commit de topo | `pull_request_read` `method: get_check_runs` (jobs do Actions e Netlify) · `get_status` (status de commit, onde aparece a Vercel) |
| Runs de um workflow | `actions_list` `method: list_workflow_runs`, `resource_id: ci.yml`, `workflow_runs_filter: {branch}` · jobs: `method: list_workflow_jobs` |
| Log do que falhou | `get_job_logs` `run_id`, `failed_only: true`, `return_content: true`, `tail_lines: 200` |
| Texto de um check que chegou por evento | `get_check_run` `checkRunId` |
| Reexecutar / disparar / cancelar | `actions_run_trigger` `method: rerun_failed_jobs` · `rerun_workflow_run` · `run_workflow` (só workflow com `workflow_dispatch`; exige `workflow_id` e `ref`) · `cancel_workflow_run` |
| Tirar do rascunho, editar, fechar | `update_pull_request` `draft: false` · `title` · `body` (substitui o corpo inteiro) · `state: closed` |
| Trazer a `main` para o PR | `update_pull_request_branch` `expectedHeadSha` |
| Mergear | `merge_pull_request` `merge_method`, `commit_title`, `commit_message`, `expectedHeadSha` |
| Revisão com comentário em linha | `pull_request_review_write` `method: create` → `add_comment_to_pending_review` → `pull_request_review_write` `method: submit_pending`, `event: COMMENT` |
| Responder e fechar comentário de revisão | `pull_request_read` `method: get_review_comments` → `add_reply_to_pull_request_comment` `commentId` → `resolve_review_thread` `threadId` (`PRRT_...`) |
| Comentário na conversa do PR | `add_issue_comment` `issue_number: <nº do PR>` |
| Branches | `list_branches` · criar: `create_branch` `branch`, `from_branch: main` · apagar: não há ferramenta; use `git push origin --delete <branch>` |
| Issue | ler: `issue_read`; criar ou fechar: `issue_write` (ao fechar, `state_reason`); listar: `list_issues` (`state: OPEN`/`CLOSED`, em maiúsculas). Não há busca de issue por texto no MCP |
| Histórico | `list_commits` `sha: <branch>`, `path` · `get_commit` · `search_commits` (maiúsculo) só procura na branch padrão |
| Tag e release | só leitura: `list_tags`, `get_tag`, `list_releases`, `get_latest_release`. Tag se cria pelo git local |
| Gravar arquivo sem clone | `push_files` (vários arquivos, 1 commit). Com clone na mão, prefira o git local |
| Vigiar o PR | `mcp__claude-code-remote__subscribe_pr_activity` / `unsubscribe_pr_activity` (`owner`, `repo`, `pullNumber`) |

Não existe ferramenta MCP para ler ou mudar a configuração do repositório, como opções de merge ou regras da `main`.

## 3. No PC, com `gh`

Não conferido se o `gh` está instalado no PC: rode `command -v gh` antes. Comandos lidos no manual do gh:

- Abrir em rascunho: `gh pr create --draft --base main --title "..." --body-file <arquivo>`. Tirar do rascunho: `gh pr ready <n>`.
- Esperar o CI do SASI: `gh run list --workflow ci.yml --branch <ramo> --limit 1 --json databaseId --jq '.[0].databaseId'` → `gh run watch <id> --exit-status`. Use o ID: sem ele, o comando falha fora de terminal interativo. Não use `gh pr checks --fail-fast` no SASI: os checks da Netlify falham sempre e encerrariam a espera.
- Log do que falhou: `gh run view --job <JOB_ID> --log-failed`. Reexecutar só o que falhou: `gh run rerun <RUN_ID> --failed`.
- Release com notas geradas dos PRs: `gh release create v0.1.0 --verify-tag --generate-notes`.

## 4. CI — trechos de YAML que valem quando mexer num workflow

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
  build-app:             # não renomear: é o nome que se confere no §5 da skill
    runs-on: ubuntu-latest
    timeout-minutes: 10  # o padrão é 360 min; o CI do SASI leva de 35 a 43 s
```

- Para guardar foto ou relatório de teste, use `actions/upload-artifact` com `if: ${{ !cancelled() }}`. Baixa-se com `actions_list` `method: list_workflow_run_artifacts` e `actions_get` `method: download_workflow_run_artifact`.
- Para reconferir um PR, reexecute o run do evento `pull_request`. Um run de `workflow_dispatch` não conta como check do PR.
