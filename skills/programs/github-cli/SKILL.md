---
name: github-cli
description: "Use a CLI `gh` (GitHub CLI) para issues, boards do GitHub Projects, PRs, releases e API — incluindo evidências com anexos de imagem via `--attach`. Use quando precisar criar/consultar/classificar tarefas no board, abrir/comentar PR, anexar prints a uma issue ou automatizar o GitHub pelo terminal."
---

# github-cli — GitHub CLI (gh) para agentes

## 1. Pré-condições (sempre)

```bash
gh auth status                     # login + token scopes
git remote get-url origin          # o repo tem remote GitHub?
```

- **Sem remote → PARE** e peça para criar/conectar o repositório. Não adivinhe.
- Scopes necessários: `repo` (issues/PR) e `project` (boards).
- `--attach` **exige write access** ao repo (ADMIN/MAINTAIN/WRITE).

## 2. Issues (tarefas)

```bash
# criar com labels (classificação tipo/prioridade)
gh issue create --repo <owner/repo> \
  --title "<título>" --body-file tarefa.md \
  --label "type:feature" --label "priority:P2"

gh issue list --repo <owner/repo> --state open --json number,title,labels
gh issue view <N> --repo <owner/repo> --comments
gh issue comment <N> --repo <owner/repo> --body-file evidencia.md
gh issue close <N> --repo <owner/repo> --comment "Resolvido por #<PR>"
```

Labels (idempotente):

```bash
gh label create "type:feature" --repo <owner/repo> --color 1d76db --description "Nova funcionalidade" --force
# type:fix|type:patch|type:refactor|type:perf|type:test|type:docs|type:chore
# priority:P0|priority:P1|priority:P2|priority:P3
```

## 3. GitHub Projects (board)

> `--owner` é **obrigatório** em contexto não interativo.

```bash
gh project list --owner <owner>
gh project field-list <n> --owner <owner>          # Status: Todo / In Progress / Done
gh project item-add <n> --owner <owner> --url <issue-url>
gh project item-list <n> --owner <owner> --format json
```

Mudar Status (use os IDs do `--format json`):

```bash
gh project item-edit --id <item-id> --project-id <project-id> \
  --field-id <status-field-id> --single-select-option-id <option-id>
```

Item sem repositório (draft):

```bash
gh project item-create <n> --owner <owner> --title "<título>" --body "<corpo>"
```

## 4. PRs

```bash
gh pr create --repo <owner/repo> --fill --body-file pr.md
gh pr checks <N>
gh pr view <N> --comments
gh pr merge <N> --squash
gh release create <tag> --generate-notes
```

## 5. Evidências com anexos (`--attach`)

- Sintaxe: `--attach './print.png#Alt text'` — imagem/vídeo, repetível (até 50), exige write.
- Se o body referencia `![alt](./print.png)`, a referência é **reescrita** para o asset uploadado.
- Vale em `gh issue create|edit|comment` e `gh pr create|edit|comment`.
- **Só imagem/vídeo**: logs longos → trecho no comentário ou `gh gist create app.log`.

```bash
gh issue comment 12 --repo astahjmo/rune --body-file evidencia.md \
  --attach ./print-1.png --attach './print-2.png#Estado final'
```

## 6. Fallback (api/graphql)

```bash
gh api repos/<owner>/<repo> --jq .node_id
gh api graphql -f query='query { viewer { login } }'
```

## 7. Regras de uso

- Criar/editar issues e itens de board **somente com confirmação** — política semiautomática do round-workflow (fora de um round aprovado; dentro dele valem os gates da skill **round-workflow**).
- Nunca deletar issue, item, label ou branch remota sem confirmação explícita.
- Prefira `--body-file` para corpos longos; markdown com referências locais + `--attach` para prints.
