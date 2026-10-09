# Round Workflow — Board, Dev com Evidências, QA Independente (Design)

- **Data:** 2026-10-09
- **Status:** Aprovado (design) — implementação pendente
- **Escopo:** `dandgabr/skills` (skills + agentes) · primeiro alvo real: `astahjmo/rune`

---

## 1. Contexto

Os subagentes de desenvolvimento hoje não têm um fluxo comum de git/GitHub: apenas
`github-specialist` e `vcs-repository-specialist` referenciam
`git-conventional-commits`/`vcs-repository-management`; não existe skill de `gh`
(issues/Projects) nem processo de triagem → board → implementação → QA. O usuário
quer que um pedido ("round") vire tarefas no board com critérios de aceitação, que um
agente dev implemente **provando com evidências** (prints/logs/repro) e que um agente
de QA independente valide com **veredito e evidências** (esperado × obtido).

## 2. Decisões

| # | Decisão |
|---|---------|
| D1 | Política **semiautomática (B)**: triagem, branch e commits locais automáticos; `push`, PR, merge e criação de issues/board somente com confirmação — o aceite do plano do round é a confirmação para criar suas tarefas. |
| D2 | **Round = o pedido da sessão atual**. A triagem classifica em `feature \| fix \| patch \| refactor \| perf \| test \| docs \| chore` + prioridade `P0–P3`. |
| D3 | O **`multi-agent-orchestrator`** conduz a pipeline e despacha dev/QA como subagentes. |
| D4 | Evidências vivem em **comentário markdown na issue/PR**, com prints inline via **`gh --attach`** (gh ≥ 2.99; v2.102.0 verificado na máquina). Logs longos: trecho no comentário ou `gh gist`. Sem binário no repo. |
| D5 | Browser dos agentes: **Browser Use CLI 2.0** (screenshots em arquivo; skill vendorizada). |
| D6 | Board **por repo**: um GitHub Project linkado ao repositório do trabalho. Para repos sem remote GitHub: criar/conectar antes de qualquer `push`/issue (com confirmação). |
| D7 | Skills novas: `github-cli`, `round-workflow`, `browser-use`. Skills de git anexadas aos agentes de dev: `git-conventional-commits`, `vcs-repository-management`. |
| D8 | Esta spec mora no repo das skills (`docs/specs/`) — a implementação é majoritariamente lá. |

## 3. Pipeline (com gates)

```
[pedido do usuário]
  1. TRIAGEM ................................. automático
     classifica tipo/prioridade; identifica repo/board; checa remote + gh auth
  2. PLANO DO ROUND .......................... [GATE: aprovação do usuário]
     propõe N tarefas (1 issue cada) com critérios de aceitação e evidências exigidas
  3. CRIAÇÃO .................................. após o gate (confirmação = aceite)
     cria labels (idempotente), issues no repo, itens no Project (Status: Todo)
  4. DISPATCH DEV (subagente) ............... automático dentro do round aprovado
     branch de trabalho; commits Conventional (locais, automáticos);
     implementa; coleta evidências; comenta na issue; card -> In Progress
  5. DISPATCH QA (subagente independente) .... automático após o dev
     reproduz e testa; comenta veredito (esperado × obtido, prints/logs);
     aprovado -> card segue; reprovado -> volta ao dev (loop <= 2 iterações;
     ao esgotar, escala ao usuário com dossiê)
  6. ENCERRAMENTO ............................. [GATE: push/PR/merge]
     push + PR (com confirmação); merge (com confirmação);
     card -> Done; issue fecha via `Closes #N`
```

**Casos de borda:**
- Sem remote GitHub → parar na triagem e pedir para criar/conectar o repo.
- Round sem código (docs/chore) → evidência = diff + execução do que der (ex.: lint).
- QA não consegue reproduzir → registrar no veredito e escalar com o passo exato que faltou.
- Bug encontrado pelo dev fora do escopo → propor issue nova (não expandir o round).

## 4. Contratos

### 4.1 Classificação do round

| Tipo | Label | Commit (Conventional) | Evidência mínima |
|------|-------|-----------------------|------------------|
| feature | `type:feature` | `feat:` | prints (UI) / logs + testes |
| fix | `type:fix` | `fix:` | repro antes/depois + testes |
| patch | `type:patch` | `fix:` (escopo patch) | diff + verificação |
| refactor | `type:refactor` | `refactor:` | testes verdes (comportamento igual) |
| perf | `type:perf` | `perf:` | medida antes/depois |
| test | `type:test` | `test:` | execução da suíte |
| docs | `type:docs` | `docs:` | diff |
| chore | `type:chore` | `chore:` | execução relevante |

Prioridade: `priority:P0` (crítico) … `priority:P3` (baixo).

### 4.2 Template de tarefa (issue)

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

### 4.3 Evidência do dev (comentário na issue)

```markdown
## ✅ Dev — <título da tarefa>
**O que mudou:** <diff resumido / arquivos>
**Como reproduzir:** <passos exatos>
**Execução:** <comandos + trecho de log>
**Prints:** (inline, via --attach)
```

Comandos:
```bash
gh issue comment <N> --repo <owner/repo> --body-file evidencia.md \
  --attach ./print-1.png --attach './print-2.png#Estado final'
# logs longos: gh gist create app.log  (linkar no comentário)
```

### 4.4 Veredito do QA (comentário na issue)

```markdown
## 🧪 QA — <resultado: ✅ PASSOU | ❌ FALHOU>
**Escopo testado:** <critérios de aceitação checados>
**Esperado × Obtido:** <por critério>
**Evidências:** prints / logs / comandos
**Riscos residuais:** <se houver>
```

### 4.5 Board e status

- Project por repo (hoje: `astahjmo/rune` — Project **#3**, linkado ao repo).
- Status: **Todo → In Progress → Done**.
  - Todo na criação; **In Progress** quando o dev começa; **Done** após QA aprovado **e** merge.
- Tipos/prioridade como **labels** (não campos custom, por simplicidade e portabilidade).

## 5. Artefatos

### 5.1 Skills novas (em `~/dandgabr-skills`)

| Caminho | Conteúdo |
|---------|----------|
| `skills/programs/github-cli/SKILL.md` | `gh auth`, issues + labels, Projects (`project item-add/item-edit/field-list`, `--owner` explícito), PRs, **`--attach`** (semântica de rewrite de referências), `gh gist`, `gh api`/GraphQL fallback. |
| `skills/engineering-practices/round-workflow/SKILL.md` | O pipeline da seção 3, contratos da seção 4, gates, loop QA, casos de borda. Fonte única do processo. |
| `skills/programs/browser-use/SKILL.md` | Skill do Browser Use (vendorizada da oficial, com atribuição): install, modos, comandos, screenshots em arquivo. |

Registro: entradas no `skills.json`, regenerar `CATALOGO.md`
(`scripts/generate_catalog.py`), `scripts/validate_skills.py` verde, symlink em
`~/.claude/skills/<name>` (padrão atual do ambiente).

### 5.2 Agentes (AGENT.md + `agent.yaml` + `agent.json`)

| Agente | Mudança |
|--------|---------|
| `multi-agent-orchestrator` | Seção "Pipeline de Round" (triagem, plano, gates, dispatch dev/QA, loop, encerramento); skills += `round-workflow`, `github-cli`, `git-conventional-commits`. |
| `qa-testing-specialist` | Protocolo de verificação independente + template de veredito + evidências via Browser Use/`gh --attach`; skills += `round-workflow`, `github-cli`, `browser-use`. |
| Devs* | Seção "Fluxo Git & GitHub (semiautomático)" + obrigação de evidência; skills += `round-workflow`, `git-conventional-commits`, `vcs-repository-management`, `github-cli` (todos) e `browser-use` apenas nos de UI: `frontend-developer`, `fullstack-developer`, `ui-ux-designer` (os demais podem usá-la por-task quando o round for de UI). |

\* Devs: `software-engineer`, `software-architect`, `frontend-developer`,
`fullstack-developer`, `java-enterprise-architect`, `code-optimizer`,
`project-reviewer`, `documenter`, `ui-ux-designer`, `github-specialist`,
`vcs-repository-specialist`.

### 5.3 Política de git dos agentes (resumo que vai na skill e nos agentes)

- **Automático:** `git status` antes de tudo; branch de trabalho; commits atômicos
  em Conventional Commits ao concluir cada etapa; nunca `--force`/reset destrutivo;
  nunca commitar segredos; não commitar mudanças não relacionadas ao round.
- **Com confirmação:** `push`, `gh pr create/merge`, criação de issues/board
  (coberta pelo aceite do plano), qualquer operação fora do round aprovado.

## 6. Gates (resumo)

| Etapa | Automático | Confirmação |
|-------|-----------|-------------|
| Triagem | ✓ | |
| Criar issues/itens no board | | ✓ (aceite do plano) |
| Branch + commits locais | ✓ | |
| Comentários de evidência (dev/QA) | ✓ (dentro do round) | |
| Movimentar card (Todo→In Progress→Done) | ✓ (dentro do round) | |
| `push` / PR / merge | | ✓ |
| Fechar issue | ✓ após merge aprovado | |

## 7. Setup de repositório

**Feito:**
- Repo privado `astahjmo/rune` criado; Project **#3 "rune"** criado e **linkado** ao repo;
- remote `origin` (`git@github.com:astahjmo/rune.git`) adicionado no clone local.

**Pendente (implementação):**
- Criar labels `type:*`/`priority:*` no repo (idempotente, pela skill `github-cli`);
- `push` inicial do código — **aguarda confirmação** (política B);
- Instalar Browser Use CLI + vendorizar a skill.

## 8. Browser Use (integração)

- CLI 2.0 (CDP direto, daemon persistente); modos: headless (default), `--headed`,
  perfil real do Chrome, `--connect` (Chrome aberto).
- Comandos-chave: `browser-use open <url>`, `state`, `click <i>`, `input`,
  `type`, `screenshot arquivo.png`, `close`.
- Uso: dev (frontend) prova a mudança; QA reproduz e coleta print/estado.
- Prints em arquivo → anexados com `gh --attach`.
- Limitações: Chrome-only; sem captcha solving local; cloud opcional ($0.02/h).

## 9. Validação

- `scripts/validate_skills.py` verde; `CATALOGO.md` regenerado; symlinks ok.
- Smoke real: 1 round pequeno no `astahjmo/rune` (ex.: um `fix` trivial de
  documentação/typo) rodando a pipeline ponta-a-ponta em modo semiautomático,
  conferindo: issue criada, item no board, commits, comentário com print, veredito
  do QA, PR com confirmação.

## 10. Riscos e não-objetivos

**Riscos:** dependência da skill oficial do Browser Use (vendorizada para mitigar);
`gh --attach` exige write access; sessão real do Chrome pode exigir interação
(login/2FA) — usar headless por padrão; loop dev↔QA limitado para evitar custo.

**Não-objetivos (agora):** automação total sem gates; board agregador cross-repo;
CI/GitHub Actions; notificações; sincronização automática das skills.

## 11. Rollout (visão; o plano detalhado vem do writing-plans)

1. Criar as 3 skills + symlinks + registro no `skills.json`/`CATALOGO.md`.
2. Atualizar agentes (orquestrador, QA, devs) + manifests.
3. Setup do repo (`astahjmo/rune`): labels; primeiro `push` com confirmação.
4. Instalar Browser Use CLI + skill.
5. Smoke test real de um round pequeno.
