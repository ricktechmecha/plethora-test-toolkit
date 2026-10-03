# Nivel 3 — Testear los Tests (No Negociable)

> "Si el mismo agente escribe el código y su prueba, la prueba tiende a codificar el bug." — Uncle Bob

**Este nivel es NO NEGOCIABLE** en un harness con agentes de IA. Sin esto, todo lo demás
es teatro.

---

## Tabla de Contenidos

| Test | Tool |
|------|------|
| [3.1 Mutation testing](#31-mutation-testing) | Stryker / mutmut / PIT |
| [3.2 Patch coverage](#32-patch-coverage) | Jest / pytest-cov |
| [3.3 Flaky test detection](#33-flaky-test-detection) | Custom / flake-nifi |
| [3.4 Assertion-free test detection](#34-assertion-free-test-detection) | ESLint custom |
| [3.5 Test-suite runtime budget](#35-test-suite-runtime-budget) | Custom |
| [3.6 Coverage ratchet](#36-coverage-ratchet) | Custom |

---

## 3.1 Mutation testing

### Qué es

Muta el código fuente (cambia `>=` por `>`, `&&` por `||`, `+` por `-`) y verifica si los
tests detectan la mutación. Si los tests siguen pasando con código mutado, los tests son
insuficientes.

### Por qué importa

**El mutation score es la métrica que audita al agente que escribe tests.** Si un agente
escribe un test que no detecta mutaciones, el test es inútil. 100% coverage con 0%
mutation score = tests que no prueban nada.

### Stack

- **Node.js / TypeScript**: [Stryker](https://github.com/stryker-mutator/stryker-js)
- **Python**: [mutmut](https://github.com/boxed/mutmut) o [cosmic-ray](https://github.com/sixty-north/cosmic-ray)
- **Java**: [PIT](https://github.com/hcoles/pitest)
- **Go**: [go-mutesting](https://github.com/zimmski/go-mutesting)

### Cómo implementarlo

```bash
# Node.js / TypeScript
npm install -D @stryker-mutator/jest-runner

# Python
pip install mutmut
```

### Config

```json
// stryker.config.json
{
  "testRunner": "jest",
  "mutate": [
    "src/services/**/*.ts",
    "src/utils/**/*.ts"
  ],
  "coverageAnalysis": "off",
  "thresholds": { "high": 80, "low": 60, "break": 60 },
  "timeoutMS": 60000,
  "concurrency": 2,
  "reporters": ["html", "clear-text", "progress"],
  "ignorePatterns": ["src/tests/**", "src/**/*.test.ts"]
}
```

### Ejecución

```bash
# Node.js
npx stryker run

# Python
mutmut run
mutmut results
mutmut show <id>
```

### Ejemplo de mutación

```typescript
// Código original
function calculateOrderTotal(items: OrderItem[]): number {
  return items.reduce((acc, item) => {
    if (item.type === 'DISCOUNT') return acc - item.amount;
    return acc + item.amount;
  }, 0);
}

// Stryker muta a:
// Mutación 1: >= en lugar de ===
if (item.type >= 'DISCOUNT') return acc - item.amount;
// Mutación 2: + en lugar de - (en DISCOUNT)
return acc + item.amount; // (en DISCOUNT branch)
// Mutación 3: remueve el if
return acc + item.amount; // (ignora DISCOUNT)
// Mutación 4: 0 en lugar de acc
return 0 + item.amount;

// Si los tests siguen pasando con estas mutaciones → tests insuficientes
```

### Cuándo usarlo

Siempre que un agente de IA escribe tests. El mutation score debe ser > 60% para lógica
de negocio crítica. Corre mutation testing en el PR gate (subset crítico) y nightly (full
suite).

---

## 3.2 Patch coverage

### Qué es

No solo "cobertura de líneas". Múltiples niveles:
- **Línea:** ¿se ejecutó esta línea?
- **Rama:** ¿se tomó este branch?
- **Condición/Decisión:** ¿cada condición booleana fue true y false?
- **Patch coverage:** ¿el diff del PR está cubierto? (lo más importante)

### Por qué importa

100% coverage global no significa que el código nuevo está cubierto. El patch coverage
mide específicamente las líneas nuevas o modificadas.

### Stack

- **Node.js**: Jest `--coverage` + `--changedSince`
- **Python**: `pytest-cov` + `diff-cover`

### Cómo implementarlo

```bash
# Node.js — coverage solo del diff
git diff --name-only main...HEAD | grep '\.ts$' > changed-files.txt
npx jest --coverage --collectCoverageFrom=$(cat changed-files.txt | tr '\n' ',')

# O usar --changedSince
npx jest --coverage --changedSince=main

# Python — diff coverage
pip install diff-cover
pytest --cov=src --cov-report=xml
diff-cover coverage.xml --compare-branch=main --html-report=diff-cover.html
```

### Ejemplo de script

```bash
#!/bin/bash
# scripts/patch-coverage.sh
CHANGED_FILES=$(git diff --name-only main...HEAD | grep 'src/.*\.ts$' | grep -v test)
if [ -z "$CHANGED_FILES" ]; then
  echo "No files to check"
  exit 0
fi

COVERAGE=$(npx jest --coverage --collectCoverageFrom="$CHANGED_FILES" 2>&1)
LINES=$(echo "$COVERAGE" | grep "All files" | awk '{print $4}' | tr -d '%')

echo "Patch coverage: $LINES%"
if [ "$LINES" -lt 80 ]; then
  echo "FAIL: Patch coverage below 80%"
  exit 1
fi
```

### Cuándo usarlo

En cada PR. El patch coverage es la métrica más importante para código nuevo — si el
código nuevo no está cubierto, la cobertura global es irrelevante. Umbral recomendado:
> 80% de líneas nuevas.

---

## 3.3 Flaky test detection

### Qué es

Tests que a veces pasan y a veces fallan sin cambios en el código. Envenenan la confianza
en toda la suite.

### Por qué importa

Un agente sin supervisión produce tests flaky por: dependencias de orden, timing, estado
compartido, I/O no mockeado. Un test flaky hace que los desarrolladores ignoren los
failures — "es flaky, pasa de nuevo" — y los bugs reales se esconden detrás.

### Cómo implementarlo

```bash
#!/bin/bash
# scripts/detect-flaky.sh — correr cada test N veces
TEST_PATTERN=${1:-"tests/unit"}
RUNS=${2:-5}
FAIL_COUNT=0

for i in $(seq 1 $RUNS); do
  echo "=== Run $i/$RUNS ==="
  RESULT=$(npx jest --testPathPattern="$TEST_PATTERN" 2>&1)
  if echo "$RESULT" | grep -q "FAIL"; then
    FAIL_COUNT=$((FAIL_COUNT + 1))
    echo "FLAKY: Run $i failed"
  fi
done

if [ "$FAIL_COUNT" -gt 0 ] && [ "$FAIL_COUNT" -lt "$RUNS" ]; then
  echo "⚠️  FLAKY TEST DETECTED: $FAIL_COUNT/$RUNS runs failed"
  exit 1
fi
```

### Script para orden aleatorio

```bash
# Detectar dependencias de orden entre tests
npx jest --testPathPattern=tests/unit --shuffle 2>&1

# Python
pytest --randomly-seed=last tests/unit/
```

### Cuándo usarlo

Nightly o semanal. Corre cada test 5-10 veces. Si cualquier test falla en al menos una
ejecución pero pasa en otras, es flaky y debe aislarse o arreglarse.

---

## 3.4 Assertion-free test detection

### Qué es

Tests que ejecutan código pero no afirman nada (`expect()` / `assert`). Un agente de IA
los produce con frecuencia.

### Por qué importa

Un test sin assertions da falso sentido de cobertura. "Pasa" pero no prueba nada. 100%
coverage con tests sin assertions es peor que 50% coverage con tests que sí afirman.

### Cómo implementarlo

```javascript
// eslint-custom-rules/no-test-without-assertion.js
module.exports = {
  meta: { type: 'problem', schema: [] },
  create(context) {
    let hasExpect = false;
    let itNode = null;

    return {
      CallExpression(node) {
        if (node.callee.name === 'it' || node.callee.name === 'test') {
          itNode = node;
          hasExpect = false;
        }
        if (node.callee.name === 'expect') {
          hasExpect = true;
        }
      },
      'CallExpression:exit'(node) {
        if (node === itNode && !hasExpect) {
          context.report({
            node,
            message: 'Test block must contain at least one expect() assertion',
          });
        }
      },
    };
  },
};
```

```javascript
// .eslintrc.json
{
  "plugins": ["custom-rules"],
  "overrides": [{
    "files": ["**/*.test.ts"],
    "rules": {
      "custom-rules/no-test-without-assertion": "error"
    }
  }]
}
```

**Python equivalente** (pytest plugin):

```python
# conftest.py
def pytest_collection_modifyitems(items):
    for item in items:
        # Check if test function has any assert statements
        source = item.function.__code__.co_code
        # Simple heuristic: check if 'assert' appears in source
        import inspect
        src = inspect.getsource(item.function)
        if 'assert ' not in src and 'pytest.raises' not in src:
            item.add_marker(pytest.mark.xfail(reason="No assertions found"))
```

### Cuándo usarlo

En el lint gate. Cada test debe tener al menos un assertion. Si un test solo ejecuta
código sin verificar nada, no es un test — es una ejecución.

---

## 3.5 Test-suite runtime budget

### Qué es

Un agente sin límite escribe 4,000 tests lentos. Necesitas un presupuesto de tiempo.

### Por qué importa

Una suite de tests que tarda 30 minutos no se corre. Si tarda más de 5 minutos, los
desarrolladores empiezan a saltársela. El budget fuerza optimización.

### Cómo implementarlo

```bash
#!/bin/bash
# scripts/test-budget.sh
START=$(date +%s)
npx jest --testPathPattern=tests/unit 2>&1
END=$(date +%s)
DURATION=$((END - START))

echo "Test suite duration: ${DURATION}s"
if [ "$DURATION" -gt 300 ]; then
  echo "FAIL: Test suite exceeds 5 minute budget (${DURATION}s)"
  exit 1
fi
```

### Config Jest

```javascript
// jest.config.js
module.exports = {
  testTimeout: 30000,     // max 30s por test individual
  maxWorkers: '50%',      // paralelización
  slowTestThreshold: 5,   // reportar tests que tardan > 5s
};
```

### Detección de tests lentos

```bash
# Jest: reportar tests lentos
npx jest --testPathPattern=tests/unit --verbose 2>&1 | grep -E '\d+s'

# Python: pytest con duración
pytest --durations=10 tests/unit/
```

### Cuándo usarlo

En el PR gate. Budget recomendado: unit tests < 3 min, integration tests < 5 min, E2E
tests < 10 min. Si un test tarda > 5s individualmente, es candidato a optimización o
reclasificación (de unit a integration).

---

## 3.6 Coverage ratchet

### Qué es

Un mecanismo que asegura que la cobertura **solo sube, nunca baja**. Guarda el valor
actual y falla si el nuevo valor es menor.

### Por qué importa

Sin ratchet, la cobertura degrada con el tiempo. Un desarrollador agrega código sin
tests, la cobertura baja 1%, nadie lo nota. En 6 meses, la cobertura bajó 20%.

### Cómo implementarlo

```javascript
// scripts/coverage-ratchet.js
const fs = require('fs');
const { execSync } = require('child_process');

const RATCHET_FILE = '.coverage-ratchet.json';

// Leer valor guardado
let previousCoverage = {};
if (fs.existsSync(RATCHET_FILE)) {
  previousCoverage = JSON.parse(fs.readFileSync(RATCHET_FILE, 'utf8'));
}

// Correr coverage y obtener resultado
const output = execSync('npx jest --coverage --json', { encoding: 'utf8' });
const result = JSON.parse(output);
const currentCoverage = result.coverageThreshold.global;

// Comparar
let failed = false;
const metrics = ['lines', 'branches', 'functions', 'statements'];
for (const metric of metrics) {
  const prev = previousCoverage[metric] || 0;
  const curr = currentCoverage[metric] || 0;
  console.log(`${metric}: ${curr}% (previous: ${prev}%)`);
  if (curr < prev) {
    console.error(`❌ ${metric} coverage decreased from ${prev}% to ${curr}%`);
    failed = true;
  }
}

if (failed) {
  process.exit(1);
}

// Guardar nuevo valor (solo si subió)
const newRatchet = {};
for (const metric of metrics) {
  newRatchet[metric] = Math.max(previousCoverage[metric] || 0, currentCoverage[metric] || 0);
}
fs.writeFileSync(RATCHET_FILE, JSON.stringify(newRatchet, null, 2));
console.log('✅ Coverage ratchet updated');
```

### Cuándo usarlo

En el quality gate. El ratchet garantiza que la cobertura es monótonamente creciente.
Cada vez que la cobertura sube, el ratchet se actualiza automáticamente.

---

## Skills y GitHub Projects

### BMad Skills relevantes

- **`bmad-review-adversarial-general`** — revisión cínica de los tests existentes. Útil
  para encontrar tests sin assertions, tests que confirman bugs, y tests flaky. El
  reviewer cínico asume que cada test es sospechoso hasta que demuestra lo contrario.
- **`bmad-editorial-review-structure`** — revisión estructural de la suite de tests.
  Útil para identificar tests duplicados, tests mal organizados, y tests que deberían
  ser parameterized.

### GitHub Projects y npm packages

| Tool | Repo / Package | Instalación |
|------|----------------|-------------|
| Stryker | [stryker-mutator/stryker-js](https://github.com/stryker-mutator/stryker-js) | `npm i -D @stryker-mutator/jest-runner` |
| mutmut (Python) | [boxed/mutmut](https://github.com/boxed/mutmut) | `pip install mutmut` |
| cosmic-ray (Python) | [sixty-north/cosmic-ray](https://github.com/sixty-north/cosmic-ray) | `pip install cosmic-ray` |
| PIT (Java) | [hcoles/pitest](https://github.com/hcoles/pitest) | Maven/Gradle plugin |
| diff-cover (Python) | [Bachmann1234/diff_cover](https://github.com/Bachmann1234/diff_cover) | `pip install diff-cover` |
| pytest-randomly | [pytest-dev/pytest-randomly](https://github.com/pytest-dev/pytest-randomly) | `pip install pytest-randomly` |
