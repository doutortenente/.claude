---
name: github-senior
description: Fluxo de git e GitHub de engenheiro sênior nos repositórios do operador (SASI, claude-workspace, .claude) — branch, commit, push, PR, CI, merge, revert, tag. Use antes de qualquer commit, push, abertura ou merge de PR, ao esperar ou ler o CI, ao desfazer mudança já publicada, e quando o operador disser "sobe", "manda pro GitHub", "abre PR", "faz o merge", "commita", "reverte" ou "o CI quebrou".
---

# GitHub como engenheiro sênior

**SASI e .claude:** nada entra na `main` sem quatro coisas: branch próprio, diff relido inteiro, check do repositório verde no **último** commit do PR (§5) e descrição que diz o porquê.

**claude-workspace:** o `CLAUDE.md` §6 de lá permite commit direto na `main`. Reler o diff inteiro continua obrigatório. Na nuvem a sessão exige branch e PR em rascunho, então vale o ciclo do §2.

Receitas de git, mapa de ferramentas e trechos de YAML: `referencia.md`, nesta pasta.

## 1. Onde você está

| Ação | Sessão na nuvem | PC do operador |
|---|---|---|
| Commit, push, tag, apagar branch | `git` no clone `/home/user/<repo>` | `git` em `~/src/claude-workspace` e em `~/.claude` (esse repo é a própria pasta). SASI: antes de usar git, rode `git -C <pasta> rev-parse --show-toplevel`; o mapa `context/workspace-claude.md` diz que `~/src/sasi` não é git |
| PR, CI, merge, revisão | ferramentas `mcp__github__*` (carregar com ToolSearch) | `gh`, se `command -v gh` achar; senão MCP |
| Esperar CI ou comentário | `mcp__claude-code-remote__subscribe_pr_activity`: o evento chega sozinho, sem consultar em laço. Se o retorno disser que outro agente já vigia o PR, o evento não vem: confira `get_check_runs` a cada retomada | `gh run watch <id> --exit-status` (`referencia.md` §3) |
| `gh` | só `gh api -X GET search/...` | livre |
| `npm` | direto | o node vem do nvm: `bash -ic 'npm run <script>'` |

Na nuvem o container morre no fim da sessão, e o reflog (diário local de onde o branch esteve) morre junto. Commit que não foi enviado com `git push` se perde.

## 2. Ciclo de uma entrega

1. **Começar.** Rode `git fetch origin` e `git status -sb`.
   - Na nuvem o branch é um só por sessão (`claude/<nome>`). Para saber se o PR anterior dele já foi mergeado: `mcp__github__list_pull_requests` com `head: "doutortenente:<branch>"`, `state: "all"`, `fields: ["number","merged","head"]`. Se foi, recomece da `main` (§6).
   - Dois assuntos viram dois PRs em série. Assunto novo com o PR do branch ainda aberto espera o merge dele; diga isso ao operador em 1 linha.
   - No PC: `git switch --no-track -c <tipo>/<assunto> origin/main`.
2. **Conferir local** antes do `git commit` (tabela do §7), e de novo antes do push se o diff mudou.
3. **Commitar.** Uma mudança lógica por commit, no formato do §3. Antes de cada `git commit`, leia `git diff --cached --stat` e `git diff --cached`. Use `git add <caminho>`; `git add -A` só depois de ler o `git status --short`.
4. **Enviar.** `git push -u origin <branch>`. Se houver erro de rede, tente de novo até 4 vezes, esperando 2, 4, 8 e 16 s. Se a conferência falhou, não abra PR. Na nuvem, envie o branch mesmo assim: commit não enviado se perde.
5. **Abrir o PR em rascunho.** Use `mcp__github__create_pull_request` com `draft: true` e `base: "main"`, com corpo no formato do §4. Na nuvem, chame `subscribe_pr_activity` logo depois.
6. **Autorrevisar.**
   - Chame `mcp__github__pull_request_read` com `method: get_files` e depois com `get_diff`. Procure arquivo que não devia estar ali, sobra de depuração e documento desatualizado.
   - Apagou ou moveu arquivo? Rode `git grep -lF "<nome-do-arquivo>"` nos três clones. Corrija todo arquivo que ainda o cita: no mesmo PR, ou num PR no outro repo. Depois rode o `git grep` de novo: ele tem de sair vazio.
7. **Passar pelo portão do CI** (§5).
8. **Tirar do rascunho.** Use `mcp__github__update_pull_request` com `draft: false`. O merge segue o §6: no SASI e no .claude, pare aqui e relate (§9) até vir a ordem.
9. **Fechar.** Chame `unsubscribe_pr_activity`.
   - No SASI, confira que o run de `ci.yml` do commit de merge na `main` (`workflow_runs_filter: {branch: "main", event: "push"}`) terminou `success`, e ponha o link no relato. Se ficou vermelho, relate na hora e proponha o revert por PR.
   - No PC, rode `git switch main && git pull --ff-only && git fetch --prune`. Na nuvem, `git fetch origin` basta, e o próximo trabalho segue o §6.

## 3. Commit

- **Título.** Formato `tipo(escopo): o que mudou`, com `tipo` ∈ `feat` · `fix` · `refactor` · `docs` · `chore` e escopo obrigatório (`CLAUDE.md` do claude-workspace §6). Sem `!` antes do `:`. Escreva como ordem curta. O git recomenda até 50 caracteres: conte **antes** de commitar com `printf %s '<título>' | LC_ALL=C.UTF-8 wc -m` (o `wc -c` conta cada acento como 2) e, se passar, encurte.
- **Corpo.** Diz o problema e por que esta abordagem; o diff já mostra o quê. Use um segundo `-m` ou, se for longo, `git commit -F <arquivo-no-scratchpad>`. `-m` e `-F` não se misturam.
- **Rodapés.** Os que a sessão exigir (`Co-Authored-By:`, `Claude-Session:`) vão juntos, um por linha, num único `-m` final, ou no fim do arquivo do `-F`. Se cada rodapé for num `-m`, vira um parágrafo, e só o último parágrafo conta como rodapé.
- **Mudança que quebra quem usa** leva no corpo a linha `BREAKING CHANGE: <o que deixa de funcionar>`, com essas duas palavras em inglês e maiúsculas.
- **Um assunto por commit.** Refatoração vai separada de correção e de recurso. Dois assuntos são dois PRs.
- **Não funcionam sem terminal, não use:**
  - `git rebase -i` sem `GIT_SEQUENCE_EDITOR=:`;
  - `git add -p` e `git add -i`;
  - `git commit` sem `-m`/`-F`, e `git commit -c`;
  - `git merge --continue` e `git rebase --continue` sem `GIT_EDITOR=true`.

  Na nuvem eles falham em silêncio. Onde houver editor configurado, ficam esperando. As alternativas estão na `referencia.md` §1.

## 4. Pull request

- **Tamanho.** Uma mudança autocontida. O guia de engenharia do Google diz que 100 linhas costuma ser razoável e 1000 costuma ser grande demais. Meça com `git diff --shortstat origin/main...HEAD`. Se passar de algumas centenas de linhas adicionadas, divida, nesta ordem e cada parte num PR: reorganização (`refactor`) → banco → tela.
- **Exceções do SASI** (`CLAUDE.md` de lá):
  - §4: mudou campo ou opção da ficha (`Ficha.tsx`), limiar ou função de gravação? A migração que atualiza `fn_ficha_contrato`, `fn_trg_janela_limiares` e o texto do manual do banco vai no mesmo PR. Sem ela, não mergeie. Aplicar a migração no banco vivo espera `[ APROVAR ]`.
  - §7: a anatomia de `sasi-ingest-export` e `admissao-uti` muda nas duas, num PR do claude-workspace, onde as duas moram. O template que esse §7 cita foi apagado (commit `1fedbd3`); diga isso em 1 linha no relato.
- **Título.** No mesmo formato do commit. No squash, só ele fica no histórico: confira com `pull_request_read` `method: get` antes do merge e corrija com `update_pull_request` `title`.
- **Corpo.** Se houver template, siga os títulos dele e preencha cada um; nenhuma seção fica vazia. No SASI:
  - "Tipo de mudança": marque a caixa do tipo do título.
  - "Como testar": os passos reais.
  - "Imagens": o MCP não anexa arquivo; escreva o motivo da falta.

  Acrescente `## Como desfazer` (ex.: "Revert do PR"). Uma caixa `[x]` só é marcada com prova: link do run ou saída do comando. Sem prova, ela fica `[ ]` com o motivo. Para ligar a uma issue, use `Closes #N`, que só fecha a issue em PR para a `main`. Proibido corpo como "ajustes", "fase 1" ou "fix bug".
- **Editar o corpo.** `update_pull_request` com `body` troca o texto inteiro. Leia antes com `pull_request_read` `method: get` e reenvie o corpo atual mais o acréscimo.
- **Rascunho.** O PR fica em rascunho enquanto falta CI ou autorrevisão. Se vier correção grande depois de pronto, volte com `draft: true`.

## 5. Portão do CI

CI (integração contínua) é a conferência automática que o GitHub roda a cada envio.

**Repo sem check na tabela do §7** (claude-workspace, .claude): o portão é a conferência local do §7, com a saída colada no corpo do PR. Não espere evento de CI.

**Repo com check (SASI):**

1. Pegue o commit de topo: `pull_request_read` `method: get` e guarde `head.sha`.
2. A `main` andou desde que o branch saiu? Rode `git fetch origin && git merge-base --is-ancestor origin/main <head.sha>`; saída 1 quer dizer que sim. Não confie em `mergeable_state: behind`. Se andou:
   - Use `mcp__github__update_pull_request_branch` com `expectedHeadSha` e volte ao passo 1.
   - Com conflito (`mergeable_state: "dirty"`), a ferramenta não resolve. No clone: `git merge --no-edit origin/main` → resolver → `git add <arquivos>` → `git commit --no-edit` → `git push`.
3. Confira os checks: `pull_request_read` `method: get_check_runs`. Vale **só o `build-app`**, que precisa estar `status: completed` e `conclusion: success`. Se estiver `queued` ou `in_progress`, espere. Job `skipped` não prova nada.
4. Os 3 checks da Netlify (`Redirect rules - sasi-uti`, `Header rules - sasi-uti`, `Pages changed - sasi-uti`) e o status `netlify/sasi-uti/deploy-preview` dão `failure` em todo commit. Por isso o `state` geral de `get_status` sai `failure` mesmo com tudo certo. Nada disso é o CI, nem barra nem libera o merge. `Vercel Preview Comments` também aparece e também não é o CI.
5. **Falhou?** Leia o log com `mcp__github__get_job_logs`, `job_id: <id do check>`, `return_content: true`. O `run_id` está no `html_url`, em `/actions/runs/<run_id>/job/`.
   - Erro de código se corrige com commit novo no branch.
   - Se o arquivo do erro não está em `git diff --name-only origin/main...HEAD`, olhe o último run da `main`. Se ela também falha, a correção é um PR próprio a partir da `main`, com ordem própria. Depois disso, o PR atual recebe `git merge origin/main`.
   - Reexecutar (`mcp__github__actions_run_trigger` `method: rerun_failed_jobs`, `run_id`) só serve para falha de rede ou da máquina do GitHub, porque roda de novo o mesmo commit.
6. **Nunca** faça commit vazio para disparar CI, não pule nem desligue teste, e não mergeie olhando o check de um commit anterior.

## 6. Merge e depois do merge

- **Quem manda mergear.**
  - SASI e .claude: só com ordem do operador na conversa ("merge", "pode juntar", `[ APROVAR ]` da entrega). No SASI, a `main` é o que a Vercel publica.
  - claude-workspace: sai sem perguntar, depois da autorrevisão e da conferência local (`docs/DECISIONS.md` de lá, decisão 6).
  - "Sobe" e "manda pro GitHub" levam até o PR pronto, não ao merge. No SASI, "deploy" é gravar no banco (`CLAUDE.md` §2 de lá), não merge.
- **Até onde vale a ordem.**
  - A ordem vale para o assunto aprovado. Dada com o CI pendente, vale até o merge: espere o evento e mergeie sem pedir de novo.
  - Commit de correção no mesmo assunto, ou um `git merge origin/main`, mantém a ordem, desde que passe de novo pelo §2 passo 6 e pelo §5.
  - Commit que acrescenta outro código, banco ou workflow exige ordem nova.
  - Ordem com o `build-app` vermelho ou pendente: diga em 1 linha qual check falhou e o que isso quebra (no SASI, a publicação). Só mergeie se ele repetir a ordem.
- **Como.** Use `mcp__github__merge_pull_request` com `merge_method: "squash"`, `commit_title: "<título do PR> (#N)"` e `expectedHeadSha: <head.sha conferido no §5>`. Se entrou commit depois da conferência, o merge é recusado: volte ao §5.
- **Branch com PR aberto não se reescreve.** Correção vai em commit novo. Para atualizar com a `main`, use `git merge origin/main`. O squash junta tudo no fim.
- **Recomeçar o branch depois do merge.** Base: o `head.sha` do PR mergeado (`pull_request_read` `method: get`, com `merged: true`).
  1. `git fetch origin`. O que sobrou fora do PR: `git log --oneline <head.sha>..HEAD`.
  2. Nada sobrou: `git switch --no-track -C <branch> origin/main`. Sobrou: `git rebase --onto origin/main <head.sha> <branch>`. Prova: `git log --oneline origin/main..HEAD` lista só o que sobrou.
  3. Envio: veja `git ls-remote --heads origin <branch>`.
     - Vazio (o normal no SASI e no claude-workspace, onde o GitHub apaga o branch no merge): `git push -u origin <branch>`.
     - Mostra o `head.sha` (o normal no .claude, onde o branch fica): `git push -u --force-with-lease=<branch>:<head.sha> origin <branch>`. Isso troca só um branch cujo conteúdo inteiro já está na `main`.
     - Outro SHA: pare. Liste o que sairia (`git log --oneline origin/main..origin/<branch>`), diga o que se perde e espere o "prosseguir".
- **Fechar PR, apagar branch, `git stash drop`, `git worktree remove`** seguem a regra global de apagar: plano com a lista completa e espera do "prosseguir".

## 7. Por repositório (medido em 26-nov-2026)

| | SASI | claude-workspace | .claude |
|---|---|---|---|
| A `main` é | o que a Vercel publica em sasi-uti.vercel.app (`CLAUDE.md` §3) | skills e config; commit direto permitido (§6) | a própria pasta `~/.claude` |
| Check que conta | `build-app` (`.github/workflows/ci.yml`: `npm ci` + `npm run build`, de 32 a 57 s) | nenhum em push ou PR; `plantao.yml` e `plantao-processar.yml` são só manuais | nenhum |
| Conferir local | Diff só de `.md`: nada a rodar. Senão: `npm ci` se não houver `node_modules` (na nuvem não há) → `npm run typecheck` → `npm run build`. Mexeu em tela: sequência do README, seção Rodar (build com as variáveis falsas, `next start -p 3111`, `node e2e/falso.cjs`, `npm run test:e2e`). Sem Playwright, a falha é de ambiente, não de código: a caixa fica `[ ]` com o motivo. Mexeu em `mcp-server/`: `npm ci && npm run build` lá dentro, porque o CI não compila essa pasta | cabeçalho de todo `SKILL.md` alterado (comando abaixo) e as skills `audit-repository` e `verify-before-finish`. Na nuvem, troque `/home/dr/src/claude-workspace` por `/home/user/claude-workspace`; o item do índice sai `[SEM_FONTE]` | cabeçalho de todo `SKILL.md` alterado (comando abaixo) |
| Template de PR | `.github/PULL_REQUEST_TEMPLATE.md` | não tem | não tem |
| Branch remoto depois do merge | apagado pelo GitHub | apagado pelo GitHub | fica |
| Como mergeava até hoje (histórico; a regra é squash, §6) | merge commit | misto: squash e merge commit | nunca teve PR |

Conferir o cabeçalho de um `SKILL.md`:

```bash
python3 -c "import sys,yaml; d=yaml.safe_load(open(sys.argv[1]).read().split('---')[1]); assert d['name'] and d['description']" <SKILL.md>
```

## 8. Erros que já aconteceram aqui

| Medido | Regra que impede |
|---|---|
| SASI: 8 de 20 PRs (#11, #12, #13, #14, #15, #16, #18, #19) mergeados antes de o `build-app` terminar; #14 e #19 por 1 e 2 s | §5, passo 3 |
| Tempo entre abrir e mergear: mediana de 52 s no SASI e de 14 s no claude-workspace, sem nenhuma revisão antes do merge | §2, passo 6 |
| PR #11 do SASI: 26.062 linhas em 144 arquivos, misturando tela, banco e reorganização | §4, Tamanho |
| 20 dos 21 commits da `main` do SASI em 60 dias têm o mesmo título, "Merge pull request #N from …/claude/nifty-babbage-n5vrib" | §6, squash com título |
| Commit `1fedbd3` apagou `docs/_SASI_TEMPLATE_BASE_v2.md`; o `CLAUDE.md` e o `docs/MEMORY.md` do SASI e a skill `admissao-uti` do claude-workspace ainda o citam | §2, passo 6 (`git grep` nos três clones) |
| Commits fora do formato `tipo(escopo):` desde 27-set-2026 na `main`, sem contar merges: SASI 1 de 26, claude-workspace 9 de 28, .claude 0 de 5 | §3 |
| `mcp-server/` do SASI mudou em 6 PRs sem nenhuma compilação automática | §7, conferir local |

## 9. Ao relatar ao operador

- Abra pelo resultado: PR #N, link e o check do §7 (`build-app` verde ou vermelho no SASI; "sem CI" + o resultado da conferência local nos outros).
- Todo termo de programação (commit, branch, PR, CI, check, merge, squash, diff, build, `main`…) leva tradução de 1 linha no próprio lugar da 1ª aparição, nunca num glossário à parte. Toda sigla é escrita por extenso.
- Não liste o que leu ou carregou, e não devolva comando para ele digitar.
