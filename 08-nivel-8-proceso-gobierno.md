# Nivel 8 — Proceso y Gobierno del Harness

> "La separación de roles es el mecanismo, no el volumen de tests." — Uncle Bob

---

## Tabla de Contenidos

| Test | Tool principal |
|------|----------------|
| [8.1 Matriz de trazabilidad](#81-matriz-de-trazabilidad) | Custom / bmad-testarch-trace |
| [8.2 Spec ambiguity review](#82-spec-ambiguity-review) | Agente |
| [8.3 Agentes adversariales](#83-agentes-adversariales-separados) | Multi-agente |
| [8.4 Definition of Done](#84-definition-of-done-y-checklist-de-pr) | Custom |
| [8.5 Exploratory testing](#85-exploratory-testing) | Manual |
| [8.6 Risk-based prioritization](#86-risk-based-test-prioritization) | Jest --findRelatedTests |
| [8.7 Quality gate + ratchet](#87-quality-gate-con-ratchet) | Custom |
| [8.8 Progressive delivery](#88-progressive-delivery) | Feature flags |
| [8.9 Post-release monitoring](#89-post-release) | Custom |
| [8.10 Métricas DORA](#810-métricas-de-resultado-dora) | Manual / script |
| [8.11 ADRs](#811-adrs-obligatorios) | MADR / log4brains |
| [8.12 Postmortem → regression](#812-postmortem--test-de-regresión-permanente) | Custom |

---

## 8.1 Matriz de trazabilidad

### Qué es
Mapeo: requisito → escenario Gherkin → test → commit. Permite auditar cobertura **de especificación**, no solo de código.

### Formato
```markdown
# docs/traceability-matrix.md

| Epic | Story | Test File | Casos | Status |
|------|-------|-----------|-------|--------|
| EPIC-1 | 1.1 | auth.test.ts | 10 | ✅ |
| EPIC-1 | 1.2 | login.spec.ts | 5 | ✅ |
| EPIC-2 | 2.1 | order.test.ts | 15 | ✅ |
| EPIC-2 | 2.2 | — | 0 | ❌ FALTA |
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| bmad-testarch-trace | (skill BMad) | Genera matriz de trazabilidad + quality gate |
| ReqIF | https://www.omg.org/spec/ReqIF/ | Estándar de requisitos |
| ReqView | https://reqview.com/ | Tool de traceability |

---

## 8.2 Spec ambiguity review

### Qué es
Un agente cuya única función es atacar la ambigüedad del requisito **antes** de que se escriba código. Barato y de altísimo rendimiento.

### Ejemplo
```
Requisito ambiguo: "El sistema debe validar orders"
→ ¿Qué validaciones? ¿Contra qué reglas? ¿Qué pasa si falla?
→ ¿Quién ve el error? ¿Se bloquea o se advierte?

Requisito desambiguado:
"El sistema debe validar que el order tenga al menos 1 item,
que el total sea positivo, y que el customerId exista en la DB.
Si falla, se rechaza con error VALIDATION_ERROR y se registra
en el log de auditoría."
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| bmad-review-adversarial-general | (skill BMad) | Cynical review / adversarial review |
| bmad-editorial-review-structure | (skill BMad) | Structural editor review |

---

## 8.3 Agentes adversariales separados

### Qué es
Separación de roles:
- **Agente A:** implementa el story
- **Agente B:** escribe el test de aceptación (Gherkin)
- **Agente C:** red-team del diff (intenta romperlo)
- **Agente D:** audita coverage y mutation score

### Por qué importa
Si el mismo agente escribe código y test, el test confirma el bug. La separación es el mecanismo.

---

## 8.4 Definition of Done y checklist de PR mecanizados

### DoD template
```markdown
## Definition of Done

- [ ] Todos los unit tests pasan
- [ ] Todos los E2E tests pasan
- [ ] Coverage thresholds cumplidos
- [ ] Mutation score > 60%
- [ ] No hay errores de TypeScript
- [ ] No hay errores de ESLint
- [ ] No hay secrets en código (gitleaks clean)
- [ ] No hay código muerto (knip clean)
- [ ] Escenarios Gherkin escritos para features nuevas
- [ ] Matriz de trazabilidad actualizada
- [ ] Threat model actualizado si hay endpoint nuevo
- [ ] Tests de seguridad para endpoints nuevos
```

### Script de verificación
```bash
#!/bin/bash
# scripts/dod-check.sh
set -e

echo "=== Definition of Done Check ==="

echo "1. TypeScript..."
npx tsc --noEmit || { echo "FAIL: TypeScript errors"; exit 1; }
echo "   ✅"

echo "2. Unit tests..."
npm run test:unit || { echo "FAIL: Unit tests failed"; exit 1; }
echo "   ✅"

echo "3. Coverage..."
npm run test:coverage || { echo "FAIL: Coverage below threshold"; exit 1; }
echo "   ✅"

echo "4. Dead code..."
npx knip || { echo "FAIL: Dead code detected"; exit 1; }
echo "   ✅"

echo "5. Secrets..."
gitleaks detect --source . --no-banner || { echo "FAIL: Secrets detected"; exit 1; }
echo "   ✅"

echo "=== All DoD checks passed ==="
```

---

## 8.5 Exploratory testing

### Qué es
Lo único que sigue requiriendo humano. Sesiones de testing exploratorio con charter definido.

### Formato
```markdown
## Session Charter
- **Duración:** 90 minutos
- **Área:** Checkout → Payment → Confirmation
- **Objetivo:** Encontrar edge cases en el flujo de pago
- **Hallazgos:**
  - [ ] Pago con tarjeta expirada → error poco claro
  - [ ] Doble submit del form → doble charge
  - [ ] Timeout del gateway → estado inconsistente
```

---

## 8.6 Risk-based test prioritization

### Qué es
Correr solo los tests afectados por el diff (test impact analysis).

### Ejecución
```bash
# Jest: solo tests relacionados con archivos cambiados
npx jest --findRelatedTests src/services/orderService.ts

# O por cambio desde main
npx jest --changedSince=main
```

---

## 8.7 Quality gate con ratchet

### Qué es
Script que ejecuta todas las verificaciones y falla si algún threshold no se cumple. Ratchet: las métricas solo suben.

### Script
```bash
#!/bin/bash
# scripts/quality-gate.sh
set -e

echo "=== Quality Gate ==="

echo "1. TypeScript strict..."
npx tsc --noEmit || exit 1

echo "2. ESLint..."
npx eslint src/ --max-warnings 0 || exit 1

echo "3. Unit tests + coverage..."
npm run test:coverage || exit 1

echo "4. Coverage ratchet..."
node scripts/coverage-ratchet.js || exit 1

echo "5. Dead code (knip)..."
npx knip || exit 1

echo "6. Secrets (gitleaks)..."
gitleaks detect --source . --no-banner || exit 1

echo "7. Duplication (jscpd)..."
npx jscpd src/ --threshold 5 || exit 1

echo "=== All gates passed ==="
```

### Coverage ratchet
```bash
#!/bin/bash
# scripts/coverage-ratchet.sh
npx jest --coverage --json --outputFile=coverage-current.json

if [ -f .coverage-floor.json ]; then
  CURRENT=$(node -e "const c=require('./coverage-current.json'); console.log(c.total.statements.pct)")
  FLOOR=$(node -e "const c=require('./.coverage-floor.json'); console.log(c.total.statements.pct)")

  echo "Current: $CURRENT% | Floor: $FLOOR%"
  if [ "$(echo "$CURRENT < $FLOOR" | bc)" -eq 1 ]; then
    echo "FAIL: Coverage decreased from $FLOOR% to $CURRENT%"
    exit 1
  fi
fi

cp coverage-current.json .coverage-floor.json
echo "Coverage floor updated to $CURRENT%"
```

---

## 8.8 Progressive delivery

### Qué es
Feature flags, canary, blue-green, shadow traffic, kill switches.

### Ejemplo (feature flags en DB)
```typescript
// FeatureFlag model
model FeatureFlag {
  id          String   @id @default(uuid())
  key         String   @unique
  enabled     Boolean  @default(false)
  rolloutPct  Int      @default(0)
  orgIds      String[]
}

// Uso
if (await featureFlagService.isEnabled('new_checkout', orgId)) {
  // nueva funcionalidad
} else {
  // funcionalidad anterior
}
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| Unleash | https://github.com/Unleash/unleash | Feature flag service open source |
| Flagsmith | https://flagsmith.com/ | Feature flags open source |
| LaunchDarkly | https://launchdarkly.com/ | Comercial |

---

## 8.9 Post-release

### Qué es
Synthetic monitoring, RUM, error budgets, crash-free session rate.

### Script de synthetic monitoring
```bash
#!/bin/bash
# scripts/synthetic-monitor.sh
HEALTH=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)
if [ "$HEALTH" != "200" ]; then
  echo "ALERT: API health check failed ($HEALTH)"
  # Enviar alerta (email, Slack, etc.)
  exit 1
fi
echo "API healthy"
```

### Crontab
```bash
# Cada 5 minutos
*/5 * * * * /opt/scripts/synthetic-monitor.sh
```

---

## 8.10 Métricas de resultado (DORA)

### Qué es
- **Lead time:** tiempo desde commit hasta producción
- **Deployment frequency:** cuántos deploys por día/semana
- **MTTR:** Mean Time To Recovery
- **Change failure rate:** % de deploys que causan incidentes

### Tracking
```bash
# scripts/dora-metrics.sh
echo "=== DORA Metrics ==="
echo "Lead time: $(git log --format='%ci' -1) → now"
echo "Deploy frequency: $(git log --oneline --since='1 week ago' | wc -l) commits this week"
echo "Change failure rate: track manually"
```

---

## 8.11 ADRs obligatorios

### Qué es
Architecture Decision Records para todas las decisiones arquitectónicas.

### Formato
```markdown
# ADR-001: Database choice

## Status
Accepted (2026-01-01)

## Context
Need to choose between PostgreSQL and MySQL for the main database.

## Decision
Use PostgreSQL for its JSON support and mature ecosystem.

## Consequences
- Need PostgreSQL expertise
- JSON queries are first-class
- Migration tooling is mature
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| MADR | https://adr.github.io/madr/ | Markdown ADR template |
| log4brains | https://github.com/thomvaill/log4brains | ADR management tool |
| ADR Tools | https://github.com/npryce/adr-tools | CLI para ADRs |

---

## 8.12 Postmortem → test de regresión permanente

### Qué es
Cada incidente genera un postmortem Y un test de regresión permanente.

### Regla
```
Incidente → Postmortem (qué pasó, por qué, cómo evitar) → Test de regresión permanente
```

### Formato de postmortem
```markdown
# Postmortem: [incident name]

## Date: [date]
## Severity: [P1/P2/P3]
## Duration: [time to resolve]

## What happened
[description]

## Root cause
[technical root cause]

## Contributing factors
[factors that made it worse]

## What went well
[things that worked]

## What went wrong
[things that failed]

## Action items
- [ ] [action item 1] (owner, due date)
- [ ] [action item 2] (owner, due date)
- [ ] Regression test for [specific bug] (owner, due date)

## Regression test
```typescript
test('regression: [bug description]', async () => {
  // Test that would have caught this bug
});
```
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| bmad-review-adversarial-general | (skill BMad) | Adversarial review que puede usarse para postmortem analysis |
