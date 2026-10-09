# Round Workflow — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar o pipeline de round (triagem → board com critérios de aceitação → dev com evidências → QA independente com veredito) no repositório de skills e agentes.

**Architecture:** 3 skills novas (`github-cli`, `round-workflow`, `browser-use`) são a fonte do processo; os agentes (`multi-agent-orchestrator`, `qa-testing-specialist`, 11 devs) referenciam essas skills e ganham uma seção de contrato no corpo. Manifestos (`agent.yaml`, `agent.json`), `skills.json` e `CATALOGO.md` ficam sincronizados. Tudo commitado em `~/dandgabr-skills`; `push` aguarda confirmação.

**Tech Stack:** Markdown + YAML/JSON (manifestos), Python 3 (scripts `validate_skills.py`/`generate_catalog.py`), `gh` 2.102+ (issues/Projects/`--attach`), Browser Use CLI 2.0.

**Spec:** `docs/specs/2026-10-09-round-workflow-design.md` (commit `f07a589`)

## Global Constraints

- Repo de trabalho: `~/dandgabr-skills` (branch `main`, sem push sem confirmação — política semiautomática).
- Todo commit usa **Conventional Commits**; não commitar mudanças fora do escopo.
- Os 3 arquivos de agente devem ficar sincronizados: `AGENT.md` (frontmatter `skills:`), `agent.yaml` (`tools: - path:`), `agent.json` (`skills:`).
- Skills novas devem ter frontmatter com `name` + `description` (exigência do `validate_skills.py`).
- `python3` no PATH com PyYAML disponível (usado pelos scripts).
- Board alvo: `astahjmo/rune` Project **#3** (`PVT_kwHOAvLIws4BmXHE`); repo privado já criado e linkado.

---

## File Structure

```
docs/plans/2026-10-09-round-workflow.md          (este plano)
skills/programs/github-cli/SKILL.md              (novo)
skills/engineering-practices/round-workflow/SKILL.md (novo)
skills/programs/browser-use/SKILL.md             (novo, vendorizado)
skills.json                                      (modificar: +3 entries)
CATALOGO.md                                      (regenerar)
~/.claude/skills/{github-cli,round-workflow,browser-use}  (symlinks)
agents/core-orchestration/multi-agent-orchestrator/{AGENT.md,agent.yaml,agent.json}
agents/software-engineering/qa-testing-specialist/{AGENT.md,agent.yaml,agent.json}
agents/software-engineering/<11 devs>/{AGENT.md,agent.yaml,agent.json}
```

---

### Task 1: Skill `github-cli`

**Files:**
- Create: `skills/programs/github-cli/SKILL.md`
- Create: `skills/programs/github-cli/{scripts,examples,resources,references}/.gitkeep`

**Interfaces:**
- Produces: skill `github-cli` — usada pelos agentes nas Tasks 6–9 e citada em `round-workflow`.

- [ ] **Step 1: Criar diretórios**

```bash
cd ~/dandgabr-skills
mkdir -p skills/programs/github-cli/{scripts,examples,resources,references}
touch skills/programs/github-cli/{scripts,examples,resources,references}/.gitkeep
```

- [ ] **Step 2: Escrever o SKILL.md com este conteúdo exato**

````markdown
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

- Criar/editar issues e itens de board **somente com confirmação** (política semiautomática do rune-workflow).
- Nunca deletar issue, item, label ou branch remota sem confirmação explícita.
- Prefira `--body-file` para corpos longos; markdown com referências locais + `--attach` para prints.
````

- [ ] **Step 3: Validar frontmatter e conteúdo**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import re
p='skills/programs/github-cli/SKILL.md'
s=open(p).read()
m=re.match(r'^---\n(.*?)\n---\n', s, re.S)
assert m, "frontmatter ausente"
assert 'name: github-cli' in m.group(1)
assert 'description:' in m.group(1)
assert 'gh project item-add' in s and '--attach' in s
print("OK github-cli")
PY
```

- [ ] **Step 4: Commit**

```bash
git add skills/programs/github-cli
git commit -m "feat(skills): add github-cli skill (issues, Projects board, PRs, --attach)"
```

---

### Task 2: Skill `round-workflow`

**Files:**
- Create: `skills/engineering-practices/round-workflow/SKILL.md`
- Create: `skills/engineering-practices/round-workflow/{scripts,examples,resources,references}/.gitkeep`

**Interfaces:**
- Consumes: `github-cli` (comandos), `git-conventional-commits` (commits).
- Produces: skill `round-workflow` — usada pelos agentes nas Tasks 6–9; define o template de tarefa/evidência/veredito.

- [ ] **Step 1: Criar diretórios**

```bash
cd ~/dandgabr-skills
mkdir -p skills/engineering-practices/round-workflow/{scripts,examples,resources,references}
touch skills/engineering-practices/round-workflow/{scripts,examples,resources,references}/.gitkeep
```

- [ ] **Step 2: Escrever o SKILL.md com este conteúdo exato**

````markdown
---
name: round-workflow
description: "Pipeline de round (o pedido da sessão): triagem em feature/fix/patch/refactor/perf/test/docs/chore, tarefas no GitHub Projects com critérios de aceitação, dev com evidências (prints/logs/repro) e QA independente com veredito esperado×obtido. Use quando um pedido deve virar issue no board com validação de QA."
---

# round-workflow — pipeline de entrega com board e QA

## 0. Classificação do round

| Tipo | Label | Commit | Evidência mínima |
|------|-------|--------|------------------|
| feature | `type:feature` | `feat:` | prints (UI) / logs + testes |
| fix | `type:fix` | `fix:` | repro antes/depois + testes |
| patch | `type:patch` | `fix:` | diff + verificação |
| refactor | `type:refactor` | `refactor:` | testes verdes (comportamento igual) |
| perf | `type:perf` | `perf:` | medida antes/depois |
| test | `type:test` | `test:` | execução da suíte |
| docs | `type:docs` | `docs:` | diff |
| chore | `type:chore` | `chore:` | execução relevante |

Prioridade: `priority:P0` (crítico) … `priority:P3` (baixo).

Regras de commit: skill **git-conventional-commits** (Conventional Commits v1.0.0).

## 1. Triagem (automática)

- Classifique o pedido (tipo + prioridade); identifique repo e Project linkado.
- Cheque `gh auth status` e `git status`; sem remote GitHub → PARE e peça para criar/conectar.

## 2. Plano do round (GATE: aprovação do usuário)

- Decomponha em N tarefas (1 issue cada). Para cada uma: título, tipo, prioridade,
  **critérios de aceitação** (checklist BDD + checks objetivos) e evidências exigidas.
- Apresente o plano; **só crie as issues/itens após o aceite**.

## 3. Criação (após o aceite)

- Labels idempotentes (`type:*`, `priority:*`), issues no repo, itens no Project (Status **Todo**).
- Comandos na skill `github-cli`.

## 4. Dispatch dev (subagente)

- Escolha o dev adequado (UI → frontend-developer; backend → software-engineer; …).
- Contrato do dev: branch de trabalho; commits na skill **git-conventional-commits** (Conventional Commits); implementar;
  **coletar evidências**; comentar na issue; card → **In Progress**.

## 5. Dispatch QA (independente)

- Subagente `qa-testing-specialist`: reproduzir pelos passos do dev, testar cada critério,
  coletar prints/logs e comentar o **veredito** (✅/❌, esperado × obtido, riscos residuais).
- ❌ → volta ao dev (máx. 2 iterações; depois escalar ao usuário com dossiê).

## 6. Encerramento

- `push`/PR/merge **somente com confirmação**. QA aprovado + merge → card **Done**; issue fecha (`Closes #N`).

## 7. Templates

### 7.1 Tarefa (issue)

```markdown
## Contexto
<pedido original do round, em 1-3 frases>

## Critérios de Aceitação
- [ ] Dado <estado>, quando <ação>, então <resultado observável>
- [ ] <check objetivo: `go test ./...` verde, lint, comportamento X>
- [ ] Sem regressão em <área afetada>

## Evidências exigidas
- Dev: prints (frontend/UI), logs de execução, passos de reprodução
- QA: veredito ✅/❌ com esperado × obtido, prints/logs

## Refs
- Round: <descrição curta do pedido>
```

### 7.2 Evidência do dev (comentário)

```markdown
## ✅ Dev — <título da tarefa>
**O que mudou:** <arquivos/diff resumido>
**Como reproduzir:** <passos exatos>
**Execução:** <comandos + trecho de log>
**Prints:** (inline, via --attach)
```

```bash
gh issue comment <N> --repo <owner/repo> --body-file evidencia.md \
  --attach ./print-1.png --attach './print-2.png#Estado final'
# logs longos: gh gist create app.log  (linkar no comentário)
```

### 7.3 Veredito do QA (comentário)

```markdown
## 🧪 QA — <✅ PASSOU | ❌ FALHOU>
**Escopo testado:** <critérios checados>
**Esperado × Obtido:** <por critério>
**Evidências:** prints / logs / comandos
**Riscos residuais:** <se houver>
```

## 8. Gates (política semiautomática)

| Etapa | Automático | Confirmação |
|-------|-----------|-------------|
| Triagem / commits locais / branch | ✓ | |
| Criar issues + itens no board | | ✓ (aceite do plano) |
| Comentários de evidência e movimento de card | ✓ (dentro do round) | |
| `push` / PR / merge | | ✓ |
````

- [ ] **Step 3: Validar**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import re
s=open('skills/engineering-practices/round-workflow/SKILL.md').read()
m=re.match(r'^---\n(.*?)\n---\n', s, re.S)
assert m and 'name: round-workflow' in m.group(1) and 'description:' in m.group(1)
for tok in ['Critérios de Aceitação','--attach','In Progress','P0','Closes #N']:
    assert tok in s, tok
print("OK round-workflow")
PY
```

- [ ] **Step 4: Commit**

```bash
git add skills/engineering-practices/round-workflow
git commit -m "feat(skills): add round-workflow skill (triagem, board, evidências, QA)"
```

---

### Task 3: Skill `browser-use` (vendorizada) + CLI

**Files:**
- Create: `skills/programs/browser-use/SKILL.md`
- Create: `skills/programs/browser-use/{scripts,examples,resources,references}/.gitkeep`

**Interfaces:**
- Produces: skill `browser-use` — usada pelos agentes de UI e pelo QA (Task 7, 8).

- [ ] **Step 1: Baixar a skill oficial (com fallback)**

```bash
cd /tmp
curl -fsSL -o browser-use-SKILL.md \
  https://raw.githubusercontent.com/browser-use/browser-use/main/skills/browser-use/SKILL.md \
  && echo "baixou oficial" || echo "SEM REDE/USAR FALLBACK"
head -20 /tmp/browser-use-SKILL.md 2>/dev/null
```

Se o download falhar, use este conteúdo mínimo no Step 2:

```markdown
# browser-use — browser para agentes de código
```bash
browser-use open <url>    # abre/atualiza a página
browser-use state         # elementos clicáveis indexados [0], [1]...
browser-use click <i>     # clique por índice
browser-use input <i> "texto"
browser-use type "texto"
# print em ARQUIVO: capture no modo Python do CLI (helpers pré-importados; ver interaction-skills/screenshots.md)
browser-use close
```
Modos: headless (default), `--headed`, `--profile "Default"` (Chrome real), `--connect` (Chrome aberto).
```

- [ ] **Step 2: Montar o SKILL.md final (oficial/fallback + uso no fluxo)**

```bash
cd ~/dandgabr-skills
mkdir -p skills/programs/browser-use/{scripts,examples,resources,references}
touch skills/programs/browser-use/{scripts,examples,resources,references}/.gitkeep
{
  echo '---'
  echo 'name: browser-use'
  echo 'description: "Browser controlado por agentes (Browser Use CLI 2.0, CDP): navegar, clicar, ler estado e tirar PRINTS em arquivo para evidências. Use quando a tarefa precisar de verificação visual/UI (frontend) ou reprodução em navegador."'
  echo '---'
  echo
  echo '<!-- Vendorizado de https://github.com/browser-use/browser-use (skills/browser-use) -->'
  echo
  cat /tmp/browser-use-SKILL.md 2>/dev/null || echo '# browser-use — comandos (ver referencia oficial)'
  echo
  echo '## Uso no nosso fluxo (evidências)'
  echo
  echo "- Prints em arquivo: capture no modo Python do CLI (\`browser-use <<'PY' … PY\`, helpers pré-importados) seguindo a referência de screenshots do browser-harness"
  echo "  (https://github.com/browser-use/browser-harness/blob/main/interaction-skills/screenshots.md); anexe com \`gh --attach\`."
  echo '- Para tarefas de UI: o dev prova o antes/depois; o QA reproduz e captura o estado.'
  echo '- Chrome-only (CDP); cloud opcional; sem captcha solving local.'
} > skills/programs/browser-use/SKILL.md
head -12 skills/programs/browser-use/SKILL.md
```

- [ ] **Step 3: Instalar o CLI e validar**

```bash
curl -fsSL https://browser-use.com/cli/install.sh | bash
export PATH="$HOME/.local/bin:$PATH"
command -v browser-use && browser-use --help | head -5
```

- [ ] **Step 4: Validar frontmatter**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import re
s=open('skills/programs/browser-use/SKILL.md').read()
m=re.match(r'^---\n(.*?)\n---\n', s, re.S)
assert m, "frontmatter"
assert 'name: browser-use' in m.group(1) and 'description:' in m.group(1)
assert 'screenshot' in s
print("OK browser-use")
PY
```

- [ ] **Step 5: Commit**

```bash
git add skills/programs/browser-use
git commit -m "feat(skills): add browser-use skill (vendored) for UI evidence screenshots"
```

---

### Task 4: Registro — `skills.json` + `CATALOGO.md` + validação

**Files:**
- Modify: `skills.json` (entries)
- Regenerate: `CATALOGO.md`

- [ ] **Step 1: Adicionar as 3 entries**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import json
p='skills.json'
d=json.load(open(p))
new=['skills/engineering-practices/round-workflow',
     'skills/programs/browser-use',
     'skills/programs/github-cli']
for e in new:
    if e not in d['entries']:
        d['entries'].append(e)
d['entries']=sorted(d['entries'])
json.dump(d, open(p,'w'), indent=2, ensure_ascii=False)
open(p,'a').write('\n')
print("entries:", len(d['entries']))
PY
```

- [ ] **Step 2: Regenerar CATALOGO.md**

```bash
python3 scripts/generate_catalog.py
git diff --stat CATALOGO.md | tail -2
```

- [ ] **Step 3: Validar (deve ficar sem errors para as novas skills)**

```bash
python3 scripts/validate_skills.py 2>&1 | tail -15
```

Esperado: sem `errors` citando `github-cli`, `round-workflow` ou `browser-use`. Warnings de subdiretório não devem existir (criamos os 4 + `.gitkeep`).

- [ ] **Step 4: Commit**

```bash
git add skills.json CATALOGO.md
git commit -m "chore(skills): register github-cli, round-workflow and browser-use in skills.json + catalog"
```

---

### Task 5: Symlinks em `~/.claude/skills`

**Files:**
- Create: `~/.claude/skills/{github-cli,round-workflow,browser-use}` → symlinks

- [ ] **Step 1: Criar e verificar**

```bash
ln -sfn ~/dandgabr-skills/skills/programs/github-cli               ~/.claude/skills/github-cli
ln -sfn ~/dandgabr-skills/skills/engineering-practices/round-workflow ~/.claude/skills/round-workflow
ln -sfn ~/dandgabr-skills/skills/programs/browser-use              ~/.claude/skills/browser-use
ls -l ~/.claude/skills/ | grep -E 'github-cli|round-workflow|browser-use'
test -f ~/.claude/skills/github-cli/SKILL.md && echo "OK symlinks"
```

- [ ] **Step 2: (sem commit — ambiente local) anotar que o opencode precisa reiniciar para carregar as skills.**

---

### Task 6: Agente `multi-agent-orchestrator`

**Files:**
- Modify: `agents/core-orchestration/multi-agent-orchestrator/AGENT.md`
- Modify: `agents/core-orchestration/multi-agent-orchestrator/agent.yaml`
- Modify: `agents/core-orchestration/multi-agent-orchestrator/agent.json`

**Interfaces:**
- Consumes: `round-workflow`, `github-cli`, `git-conventional-commits`.

- [ ] **Step 1: Rodar o script de atualização**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import os, re
base='agents/core-orchestration/multi-agent-orchestrator'
SKILLS=['../../../skills/engineering-practices/round-workflow/SKILL.md',
        '../../../skills/programs/github-cli/SKILL.md',
        '../../../skills/engineering-practices/git-conventional-commits/SKILL.md']
md=open(f'{base}/AGENT.md').read()
assert '## 🚀 Como Executar' in md, "anchor do corpo"
if '## 🎯 Pipeline de Round' not in md:
    sec = """
---

## 🎯 Pipeline de Round (board → dev → QA)

Você conduz o fluxo da skill **round-workflow** para cada pedido ("round"):

1. **Triagem** — classifique (feature/fix/patch/refactor/perf/test/docs/chore + P0..P3); confirme repo/Project e `gh auth`.
2. **Plano do round** — proponha as tarefas (1 issue cada) com critérios de aceitação e evidências exigidas; **aguarde o aceite**.
3. **Criação** — crie labels/issues/itens no board (Status Todo) usando a skill **github-cli** (após o aceite).
4. **Dispatch dev** — subagente dev adequado; contrato: branch + commits Conventional; implementar; evidenciar (prints/logs/repro); comentar na issue; card → In Progress.
5. **Dispatch QA** — subagente `qa-testing-specialist` independente; veredito ✅/❌ com esperado × obtido; falha → dev (máx. 2 iterações; depois escale com dossiê).
6. **Encerramento** — push/PR/merge somente com confirmação; QA + merge → card Done; issue fecha (`Closes #N`).

Regras: nunca trabalhe direto na `main`; nunca `push` sem confirmação; registre no board cada transição de status.

"""
    md=md.replace('## 🚀 Como Executar', sec.lstrip('\n')+'## 🚀 Como Executar',1)
# frontmatter skills
m=re.match(r'^---\n(.*?\n)---\n', md, re.S)
fm=m.group(1)
for s in SKILLS:
    if f'- {s}' not in fm:
        fm=fm.rstrip('\n')+f'\n- {s}\n'
md=md[:m.start(1)]+fm+md[m.end(1):]
open(f'{base}/AGENT.md','w').write(md)
# agent.yaml
y=open(f'{base}/agent.yaml').read()
for s in SKILLS:
    if f'- path: {s}' not in y:
        y=y.rstrip('\n')+f'\n- path: {s}\n'
open(f'{base}/agent.yaml','w').write(y)
# agent.json
import json
j=json.load(open(f'{base}/agent.json'))
for s in SKILLS:
    if s not in j['skills']:
        j['skills'].append(s)
json.dump(j, open(f'{base}/agent.json','w'), indent=2, ensure_ascii=False)
open(f'{base}/agent.json','a').write('\n')
print("orchestrator OK")
PY
```

- [ ] **Step 2: Verificar**

```bash
cd ~/dandgabr-skills/agents/core-orchestration/multi-agent-orchestrator
rg -c 'Pipeline de Round' AGENT.md
rg -c 'round-workflow|github-cli|git-conventional-commits' AGENT.md agent.yaml agent.json
python3 - <<'PY'
import re, json, yaml
fm=re.match(r'^---\n(.*?)\n---\n', open('AGENT.md').read(), re.S).group(1)
d=yaml.safe_load(fm)
assert any('round-workflow' in x for x in d['skills'])
assert any('github-cli' in x for x in d['skills'])
json.load(open('agent.json'))
print('frontmatter+json ok')
PY
```

- [ ] **Step 3: Commit**

```bash
cd ~/dandgabr-skills
git add agents/core-orchestration/multi-agent-orchestrator
git commit -m "feat(agents): multi-agent-orchestrator conducts round pipeline (board/dev/QA)"
```

---

### Task 7: Agente `qa-testing-specialist`

**Files:**
- Modify: `agents/software-engineering/qa-testing-specialist/{AGENT.md,agent.yaml,agent.json}`

**Interfaces:**
- Consumes: `round-workflow`, `github-cli`, `browser-use`.

- [ ] **Step 1: Script de atualização (mesmo padrão da Task 6)**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import re, json
base='agents/software-engineering/qa-testing-specialist'
SKILLS=['../../../skills/engineering-practices/round-workflow/SKILL.md',
        '../../../skills/programs/github-cli/SKILL.md',
        '../../../skills/programs/browser-use/SKILL.md']
md=open(f'{base}/AGENT.md').read()
assert '## 🚀 Como Executar' in md
if '## 🧪 Verificação Independente (Pipeline de Round)' not in md:
    sec = """
---

## 🧪 Verificação Independente (Pipeline de Round)

No fluxo da skill **round-workflow**, você é o QA independente de cada tarefa:

1. **Reproduza** pelos passos do dev (comentário de evidência na issue).
2. **Teste cada critério de aceitação** (checklist BDD + checks objetivos) — não confie no relato.
3. **Colete evidências**: prints (browser-use para UI), logs de execução, comandos exatos.
4. **Comente o veredito** na issue (template 7.3 da skill): ✅/❌, esperado × obtido, riscos residuais.
5. **Falha → devolve ao dev** com o erro reproduzível (máx. 2 iterações; depois escale com dossiê).

Use **github-cli** para comentar/anexar prints (`gh --attach`). Nunca aprove sem evidência verificável.

"""
    md=md.replace('## 🚀 Como Executar', sec.lstrip('\n')+'## 🚀 Como Executar',1)
m=re.match(r'^---\n(.*?\n)---\n', md, re.S); fm=m.group(1)
for s in SKILLS:
    if f'- {s}' not in fm: fm=fm.rstrip('\n')+f'\n- {s}\n'
md=md[:m.start(1)]+fm+md[m.end(1):]
open(f'{base}/AGENT.md','w').write(md)
y=open(f'{base}/agent.yaml').read()
for s in SKILLS:
    if f'- path: {s}' not in y: y=y.rstrip('\n')+f'\n- path: {s}\n'
open(f'{base}/agent.yaml','w').write(y)
j=json.load(open(f'{base}/agent.json'))
for s in SKILLS:
    if s not in j['skills']: j['skills'].append(s)
json.dump(j, open(f'{base}/agent.json','w'), indent=2, ensure_ascii=False); open(f'{base}/agent.json','a').write('\n')
print("qa OK")
PY
```

- [ ] **Step 2: Verificar + commit**

```bash
cd ~/dandgabr-skills
rg -c 'Verificação Independente' agents/software-engineering/qa-testing-specialist/AGENT.md
git add agents/software-engineering/qa-testing-specialist
git commit -m "feat(agents): qa-testing-specialist gets independent round verification protocol"
```

---

### Task 8: Devs — lote A (core)

**Agentes:** `software-engineer`, `software-architect`, `frontend-developer`, `fullstack-developer`, `java-enterprise-architect`.

**Interfaces:**
- Consumes: `round-workflow`, `git-conventional-commits`, `vcs-repository-management`, `github-cli`; `browser-use` nos de UI.

- [ ] **Step 1: Script (aplica o bloco + skills nos 3 arquivos; UI recebe browser-use)**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import re, json
BASE='agents/software-engineering'
AGENTS={'software-engineer':False,'software-architect':False,'frontend-developer':True,
        'fullstack-developer':True,'java-enterprise-architect':False}
COMMON=['../../../skills/engineering-practices/round-workflow/SKILL.md',
        '../../../skills/engineering-practices/git-conventional-commits/SKILL.md',
        '../../../skills/engineering-practices/vcs-repository-management/SKILL.md',
        '../../../skills/programs/github-cli/SKILL.md']
BROWSER='../../../skills/programs/browser-use/SKILL.md'
BLOCK="""
---

## 🔀 Fluxo Git & GitHub (Semiautomático)

Siga a skill **round-workflow** (triagem, tarefas no board com critérios de aceitação, evidências, QA independente). Regras:

- **Automático:** `git status` antes de tudo; branch de trabalho (nunca na `main`); commits atômicos em **Conventional Commits** ao concluir cada etapa; nunca `--force`/reset destrutivo; nunca commitar segredos ou mudanças fora do escopo.
- **Com confirmação:** `push`, `gh pr create/merge`, criação/edição de issues e itens no GitHub Projects.
- **Evidência obrigatória:** ao concluir a tarefa, poste o comentário de evidência na issue (prints via Browser Use + `gh --attach`, logs de execução, passos de reprodução) — prove que fez.

"""
for name, ui in AGENTS.items():
    d=f'{BASE}/{name}'
    md=open(f'{d}/AGENT.md').read()
    assert '## 🚀 Como Executar' in md, f'{name}: anchor'
    if 'Fluxo Git & GitHub (Semiautomático)' not in md:
        md=md.replace('## 🚀 Como Executar', BLOCK.lstrip('\n')+'## 🚀 Como Executar',1)
    skills=COMMON+([BROWSER] if ui else [])
    m=re.match(r'^---\n(.*?\n)---\n', md, re.S); fm=m.group(1)
    for s in skills:
        if f'- {s}' not in fm: fm=fm.rstrip('\n')+f'\n- {s}\n'
    md=md[:m.start(1)]+fm+md[m.end(1):]
    open(f'{d}/AGENT.md','w').write(md)
    y=open(f'{d}/agent.yaml').read()
    for s in skills:
        if f'- path: {s}' not in y: y=y.rstrip('\n')+f'\n- path: {s}\n'
    open(f'{d}/agent.yaml','w').write(y)
    j=json.load(open(f'{d}/agent.json'))
    for s in skills:
        if s not in j['skills']: j['skills'].append(s)
    json.dump(j, open(f'{d}/agent.json','w'), indent=2, ensure_ascii=False); open(f'{d}/agent.json','a').write('\n')
    print("OK", name, "(ui)" if ui else "")
PY
```

- [ ] **Step 2: Verificar consistência (3 arquivos por agente)**

```bash
cd ~/dandgabr-skills
for a in software-engineer software-architect frontend-developer fullstack-developer java-enterprise-architect; do
  d=agents/software-engineering/$a
  echo "== $a: $(rg -c 'round-workflow' $d/AGENT.md $d/agent.yaml $d/agent.json | tr '\n' ' ')"
  python3 -c "import json;json.load(open('$d/agent.json'))" || echo "JSON QUEBRADO $a"
done
```

- [ ] **Step 3: Commit**

```bash
git add agents/software-engineering/software-engineer agents/software-engineering/software-architect \
        agents/software-engineering/frontend-developer agents/software-engineering/fullstack-developer \
        agents/software-engineering/java-enterprise-architect
git commit -m "feat(agents): git/GitHub round flow + evidence duty for core dev agents"
```

---

### Task 9: Devs — lote B (restante)

**Agentes:** `code-optimizer`, `project-reviewer`, `documenter`, `ui-ux-designer`, `github-specialist`, `vcs-repository-specialist`.

- [ ] **Step 1: Reutilizar o script da Task 8 trocando o dict**

```bash
cd ~/dandgabr-skills
python3 - <<'PY'
import re, json
BASE='agents/software-engineering'
AGENTS={'code-optimizer':False,'project-reviewer':False,'documenter':False,
        'ui-ux-designer':True,'github-specialist':False,'vcs-repository-specialist':False}
COMMON=['../../../skills/engineering-practices/round-workflow/SKILL.md',
        '../../../skills/engineering-practices/git-conventional-commits/SKILL.md',
        '../../../skills/engineering-practices/vcs-repository-management/SKILL.md',
        '../../../skills/programs/github-cli/SKILL.md']
BROWSER='../../../skills/programs/browser-use/SKILL.md'
BLOCK = """
---

## 🔀 Fluxo Git & GitHub (Semiautomático)

Siga a skill **round-workflow** (triagem, tarefas no board com critérios de aceitação, evidências, QA independente). Regras:

- **Automático:** `git status` antes de tudo; branch de trabalho (nunca na `main`); commits atômicos em **Conventional Commits** ao concluir cada etapa; nunca `--force`/reset destrutivo; nunca commitar segredos ou mudanças fora do escopo.
- **Com confirmação:** `push`, `gh pr create/merge`, criação/edição de issues e itens no GitHub Projects.
- **Evidência obrigatória:** ao concluir a tarefa, poste o comentário de evidência na issue (prints via Browser Use + `gh --attach`, logs de execução, passos de reprodução) — prove que fez.

"""
for name, ui in AGENTS.items():
    d=f'{BASE}/{name}'
    md=open(f'{d}/AGENT.md').read()
    assert '## 🚀 Como Executar' in md, f'{name}: anchor'
    if 'Fluxo Git & GitHub (Semiautomático)' not in md:
        md=md.replace('## 🚀 Como Executar', BLOCK.lstrip('\n')+'## 🚀 Como Executar',1)
    skills=COMMON+([BROWSER] if ui else [])
    m=re.match(r'^---\n(.*?\n)---\n', md, re.S); fm=m.group(1)
    for s in skills:
        if f'- {s}' not in fm: fm=fm.rstrip('\n')+f'\n- {s}\n'
    md=md[:m.start(1)]+fm+md[m.end(1):]
    open(f'{d}/AGENT.md','w').write(md)
    y=open(f'{d}/agent.yaml').read()
    for s in skills:
        if f'- path: {s}' not in y: y=y.rstrip('\n')+f'\n- path: {s}\n'
    open(f'{d}/agent.yaml','w').write(y)
    j=json.load(open(f'{d}/agent.json'))
    for s in skills:
        if s not in j['skills']: j['skills'].append(s)
    json.dump(j, open(f'{d}/agent.json','w'), indent=2, ensure_ascii=False); open(f'{d}/agent.json','a').write('\n')
    print("OK", name, "(ui)" if ui else "")
PY
```

- [ ] **Step 2: Verificar + commit**

```bash
cd ~/dandgabr-skills
for a in code-optimizer project-reviewer documenter ui-ux-designer github-specialist vcs-repository-specialist; do
  python3 -c "import json;json.load(open('agents/software-engineering/$a/agent.json'))" || echo "JSON QUEBRADO $a"
done
python3 scripts/validate_skills.py 2>&1 | tail -5
git add agents/software-engineering/code-optimizer agents/software-engineering/project-reviewer \
        agents/software-engineering/documenter agents/software-engineering/ui-ux-designer \
        agents/software-engineering/github-specialist agents/software-engineering/vcs-repository-specialist
git commit -m "feat(agents): git/GitHub round flow + evidence duty for remaining dev agents"
```

---

### Task 10: Labels no `astahjmo/rune`

- [ ] **Step 1: Criar labels idempotentes (type + priority)**

```bash
R=astahjmo/rune
for t in feature fix patch refactor perf test docs chore; do
  gh label create "type:$t" --repo $R --color 1d76db --description "Tipo: $t" --force
done
for p in P0 P1 P2 P3; do
  gh label create "priority:$p" --repo $R --color d93f0b --description "Prioridade $p" --force
done
gh label list --repo $R | grep -E 'type:|priority:'
```

- [ ] **Step 2: (sem commit — estado remoto)**

---

### Task 11: Smoke E2E (mini-round real)

**Files:** nenhum (valida a toolchain; não altera código).

- [ ] **Step 1: Issue de teste + item no board**

```bash
gh issue create --repo astahjmo/rune --title "smoke: valida pipeline (pode fechar)" \
  --body "Teste do fluxo round-workflow. Sem código." --label "type:chore" --label "priority:P3"
# anotar o N e o URL; adicionar ao board:
gh project item-add 3 --owner astahjmo --url <issue-url>
gh project item-list 3 --owner astahjmo --format json | head -c 400
```

- [ ] **Step 2: Print real via Browser Use e comentário com `--attach`**

```bash
echo '<h1>rune round-workflow smoke ok</h1>' > /tmp/smoke.html
(cd /tmp && python3 -m http.server 8099 &) ; sleep 1
browser-use <<'PY'
new_tab("http://127.0.0.1:8099/smoke.html")
capture_screenshot("/tmp/smoke.png")
PY
pkill -f "http.server 8099" || true
cat > /tmp/evidencia.md <<'MD'
## ✅ Dev — smoke do pipeline
**Execução:** servidor local + captura via browser-use (modo Python)
**Prints:** abaixo
MD
gh issue comment <N> --repo astahjmo/rune --body-file /tmp/evidencia.md --attach /tmp/smoke.png
```

- [ ] **Step 3: Mover card para In Progress e fechar o smoke**

```bash
PROJ=PVT_kwHOAvLIws4BmXHE
get() { gh project field-list 3 --owner astahjmo --format json | python3 -c "
import json,sys
d=json.load(sys.stdin)
f=[x for x in d['fields'] if x['name']=='Status'][0]
print(f['id'] if '$1'=='field' else [o['id'] for o in f['options'] if o['name']=='$1'][0])"; }
FIELD=$(get field); OPT=$(get In Progress)
ITEM=$(gh project item-list 3 --owner astahjmo --format json | python3 -c "
import json,sys
d=json.load(sys.stdin)
print([i['id'] for i in d['items'] if 'smoke' in i.get('content',{}).get('title','')][0])")
gh project item-edit --id "$ITEM" --project-id "$PROJ" --field-id "$FIELD" --single-select-option-id "$OPT"
gh issue close <N> --repo astahjmo/rune --comment "Smoke concluído: pipeline validado."
```

- [ ] **Step 4: Reportar ao usuário** o que passou; pedir confirmação para:
  1. `git push` do repo de skills (`dandgabr/skills` — commits das Tasks 1–9);
  2. `git push -u origin main` do `rune` (não feito ainda).

---

## Self-Review (do plano)

- **Spec coverage:** skills (T1–T3), registro (T4), ambiente (T5), orquestrador (T6), QA (T7), devs (T8–T9), board/labels (T10), validação ponta-a-ponta (T11) — cobre as seções 3–9 da spec.
- **Placeholders:** comandos documentam `<N>`, `<item-id>` etc. como valores a obter via `--format json` (passos mostram como); nenhum "TODO".
- **Consistência de nomes:** `round-workflow`, `github-cli`, `browser-use`; Project `3`; skill paths `../../../skills/...` consistentes nos 3 arquivos.
