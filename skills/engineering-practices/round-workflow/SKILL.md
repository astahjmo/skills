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

## 9. Casos de borda

- Sem remote GitHub → parar na triagem e pedir para criar/conectar o repositório (já coberto na §1).
- Round sem código (docs/chore) → evidência = diff + execução do que der (ex.: lint).
- QA não consegue reproduzir → registrar no veredito e escalar com o passo exato que faltou.
- Bug encontrado pelo dev fora do escopo → propor issue nova (não expandir o round).
