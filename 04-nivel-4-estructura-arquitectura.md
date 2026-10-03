# Nivel 4 — Estructura y Arquitectura

> "Las métricas no capturan vulnerabilidades ni código muerto que un ingeniero reconoce de vista." — Grady Booch (objeción a Uncle Bob)

Métricas de estructura, acoplamiento y complejidad. Codifican las reglas de capas y
prohíben ciclos.

---

## Tabla de Contenidos

| Test | Tool |
|------|------|
| [4.1 Conformidad arquitectónica](#41-conformidad-arquitectónica) | dependency-cruiser |
| [4.2 Métricas de acoplamiento](#42-métricas-de-acoplamiento) | dependency-cruiser |
| [4.3 Complejidad](#43-complejidad) | ESLint / radon |
| [4.4 SAST / análisis estático](#44-sast--análisis-estático-profundo) | Semgrep |
| [4.5 API surface diffing](#45-api-surface-diffing--semver-compliance) | Custom / openapi-diff |
| [4.6 Migration safety](#46-migration-safety) | Custom |
| [4.7 Doc coverage](#47-doc-coverage-y-doctests-ejecutables) | ESLint / pydocstyle |
| [4.8 Análisis de churn / hotspots](#48-análisis-de-churn--hotspots) | git stats / code-complexity |

---

## 4.1 Conformidad arquitectónica

### Qué es

Verifica que el código sigue las reglas de arquitectura: capas, direcciones de
dependencia, sin ciclos.

### Por qué importa

La arquitectura se degrada silenciosamente. Un desarrollador importa `database` en un
controller "solo por esta vez" y la separación de capas se rompe. Sin verificación
automática, la degradación es irreversible.

### Reglas típicas (ejemplo genérico)

```
Routes/Controllers → Services → Repositories/DataAccess (NUNCA Routes → DataAccess directamente)
Services → Repositories (NUNCA Services → Routes)
Frontend → API (NUNCA Frontend → DataAccess/DB directamente)
No dependencias circulares entre módulos
```

### Stack

- **Node.js / TypeScript**: [dependency-cruiser](https://github.com/sverweij/dependency-cruiser)
- **Python**: [import-linter](https://github.com/seddonym/import-linter) o [pydeps](https://github.com/thebjorn/pydeps)
- **Java**: [ArchUnit](https://github.com/TNG/ArchUnit)

### Config

```javascript
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    {
      name: 'no-circular',
      severity: 'error',
      comment: 'No dependencias circulares',
      from: {},
      to: { circular: true },
    },
    {
      name: 'routes-no-data-access',
      severity: 'error',
      comment: 'Controllers no pueden importar data-access directamente',
      from: { path: 'src/routes' },
      to: { path: 'src/data-access' },
    },
    {
      name: 'services-no-routes',
      severity: 'error',
      from: { path: 'src/services' },
      to: { path: 'src/routes' },
    },
    {
      name: 'no-frontend-to-backend-internal',
      severity: 'error',
      from: { path: 'frontend/src' },
      to: { path: 'backend/src' },
    },
  ],
};
```

### Ejecución

```bash
# Node.js
npx depcruise src --config .dependency-cruiser.js

# Python
# import-linter
cat > .importlinter << 'EOF'
[importlinter]
root_package = myproject

[importlinter:contract:layers]
name = Layered architecture
type = layers
layers =
    myproject.routes
    myproject.services
    myproject.repositories
EOF
lint-imports
```

### Cuándo usarlo

Siempre que el proyecto tenga una arquitectura en capas o modular. Las reglas deben
codificarse como código, no como documentación que nadie lee.

---

## 4.2 Métricas de acoplamiento

### Qué es

Inestabilidad (I = Ce/(Ca+Ce)), abstracción (A = Na/N), distancia de la secuencia
principal (|A + I - 1|). Métricas de Robert C. Martin ("Clean Architecture").

### Por qué importa

- **Inestabilidad (I):** mide cuánto depende un módulo de otros vs. cuántos dependen de
  él. I=0 es maximamente estable (nadie depende de ti, tú dependes de muchos). I=1 es
  maximamente inestable (muchos dependen de ti, tú no dependes de nadie).
- **Abstracción (A):** ratio de tipos abstractos vs. concretos. A=0 es todo concreto,
  A=1 es todo abstracto.
- **Distancia de la secuencia principal:** |A + I - 1|. Ideal = 0. Mide si el módulo
  está balanceado entre abstracción y estabilidad.

### Stack

- **Node.js**: dependency-cruiser (genera métricas)
- **Python**: [radon](https://github.com/rubik/radon) + custom scripts

### Ejemplo

```bash
# dependency-cruiser con métricas
npx depcruise src --config .dependency-cruiser.js --output-type metrics --output-to metrics.json

# Python con radon
pip install radon
radon cc src/ -a  # complejidad ciclomática + average
radon mi src/     # maintainability index
```

### Cuándo usarlo

En el quality gate semanal. Las métricas de acoplamiento no bloquean commits pero deben
monitorearse para detectar degradación arquitectónica gradual.

---

## 4.3 Complejidad

### Qué es

Ciclomática, cognitiva, Halstead, índice de mantenibilidad, profundidad de anidamiento,
longitud de función.

### Por qué importa

Funciones con complejidad ciclomática > 10 son difíciles de testear exhaustivamente
(necesitan 2^10 = 1024 caminos). Funciones > 50 líneas suelen hacer demasiadas cosas.

### Stack

- **Node.js / TypeScript**: ESLint complexity rules
- **Python**: [radon](https://github.com/rubik/radon) o [ruff](https://github.com/astral-sh/ruff) (C901)
- **Rust**: `cargo clippy` (cognitive_complexity)

### Config ESLint

```json
// .eslintrc.json
{
  "rules": {
    "complexity": ["error", 10],
    "max-depth": ["error", 4],
    "max-lines-per-function": ["error", { "max": 80, "skipComments": true }],
    "max-statements": ["error", 20],
    "max-nested-callbacks": ["error", 3],
    "max-params": ["error", 5]
  }
}
```

**Python equivalente** (`pyproject.toml`):

```toml
[tool.ruff.lint]
select = ["C901"]  # mccabe complexity

[tool.ruff.lint.mccabe]
max-complexity = 10
```

### Cuándo usarlo

En el lint gate. Complejidad ciclomática > 10 debe ser un warning, > 15 un error.
Funciones > 80 líneas deben justificarse en el PR.

---

## 4.4 SAST / análisis estático profundo

### Qué es

Análisis estático profundo con reglas custom. Detecta vulnerabilidades, código
sospechoso, patrones prohibidos.

### Stack

- **Cualquier lenguaje**: [Semgrep](https://github.com/semgrep/semgrep)
- **Node.js**: [eslint-plugin-security](https://github.com/eslint-community/eslint-plugin-security)
- **Python**: [bandit](https://github.com/PyCQA/bandit)

### Config

```yaml
# .semgrep.yml
rules:
  - id: no-hardcoded-secrets
    pattern-regex: '(API_KEY|JWT_SECRET|PRIVATE_KEY|PASSWORD)\s*=\s*["'\'']'
    message: "Never hardcode secrets in source code"
    severity: ERROR
    languages: [typescript, javascript, python]

  - id: no-raw-sql-injection
    pattern: $DB.$queryRawUnsafe($USER_INPUT)
    message: "Use parameterized queries, not raw query with user input"
    severity: ERROR
    languages: [typescript]

  - id: no-missing-auth
    pattern: "router.$METHOD('$PATH', $HANDLER)"
    metavariable-regex:
      metavariable: $METHOD
      regex: "get|post|put|delete|patch"
    message: "Verify route has auth middleware"
    severity: WARN
    languages: [typescript]

  - id: no-eval
    pattern: eval($EXPR)
    message: "Never use eval()"
    severity: ERROR
    languages: [typescript, javascript, python]

  - id: no-weak-crypto
    pattern: crypto.createHash('md5', $DATA)
    message: "MD5 is cryptographically broken. Use SHA-256 or stronger."
    severity: ERROR
    languages: [typescript, javascript]
```

### Ejecución

```bash
# Semgrep
semgrep --config .semgrep.yml src/

# Semgrep con reglas de la comunidad + custom
semgrep --config auto --config .semgrep.yml src/

# Python con bandit
bandit -r src/ -f json -o bandit-report.json
```

### Cuándo usarlo

En el quality gate (PR o nightly). Semgrep con reglas custom detecta patrones
específicos del proyecto que los linters genéricos no conocen.

---

## 4.5 API surface diffing / semver compliance

### Qué es

Detecta cambios breaking en la API pública entre versiones.

### Stack

- **OpenAPI**: [openapi-diff](https://github.com/OpenAPITools/openapi-diff)
- **TypeScript**: [ts-api-guardian](https://github.com/GoogleChromeLabs/ts-api-guardian) o [api-extractor](https://github.com/microsoft/rushstack/tree/main/apps/api-extractor)
- **Rust**: `cargo semver-checks`

### Ejemplo

```bash
# OpenAPI diff
openapi-diff old-openapi.yaml new-openapi.yaml --format text

# TypeScript API extractor
npm install -D @microsoft/api-extractor
npx api-extractor run --local
```

### Cuándo usarlo

Cuando se publica una API (REST, GraphQL, o librería npm). El diff debe bloquear
cambios breaking en minor/patch versions y requerir major version bump.

---

## 4.6 Migration safety

### Qué es

Linter de migraciones que detecta operaciones no backward-compatible (DROP COLUMN,
ALTER TYPE, etc.).

### Cómo implementarlo

```bash
#!/bin/bash
# scripts/check-migration-safety.sh
MIGRATION=$1
if grep -qE "DROP COLUMN|DROP TABLE|ALTER COLUMN.*TYPE|RENAME COLUMN|RENAME TABLE" "$MIGRATION"; then
  echo "WARNING: Migration contains potentially breaking operation:"
  grep -nE "DROP COLUMN|DROP TABLE|ALTER COLUMN.*TYPE|RENAME COLUMN|RENAME TABLE" "$MIGRATION"
  echo "Consider a multi-step migration:"
  echo "  1. Add new column/table (backward compatible)"
  echo "  2. Deploy code that writes to both old and new"
  echo "  3. Backfill data"
  echo "  4. Deploy code that reads from new only"
  echo "  5. Drop old column/table (backward incompatible)"
  # No fallar automáticamente — a veces es necesario
fi
```

### Ejemplo de migración segura (multi-step)

```sql
-- Step 1: Add new column (backward compatible)
ALTER TABLE orders ADD COLUMN status_new VARCHAR(20);

-- Step 2: Backfill (after deploy)
UPDATE orders SET status_new = CASE
  WHEN status = 'PENDING' THEN 'PENDING_PAYMENT'
  WHEN status = 'DONE' THEN 'CONFIRMED'
  ELSE status
END;

-- Step 3: Deploy code that uses status_new

-- Step 4: Drop old column (after verification)
ALTER TABLE orders DROP COLUMN status;
ALTER TABLE orders RENAME COLUMN status_new TO status;
```

### Cuándo usarlo

En cada migración de base de datos. Las migraciones breaking deben ser multi-step para
permitir deploys sin downtime.

---

## 4.7 Doc coverage y doctests ejecutables

### Qué es

Verifica que las funciones públicas tienen documentación (JSDoc/docstrings) y que los
ejemplos en docs son ejecutables.

### Stack

- **Node.js / TypeScript**: ESLint `require-jsdoc`
- **Python**: [pydocstyle](https://github.com/PyCQA/pydocstyle) + [doctest](https://docs.python.org/3/library/doctest.html)

### Config ESLint

```json
{
  "rules": {
    "require-jsdoc": ["warn", {
      "require": {
        "FunctionDeclaration": true,
        "MethodDefinition": true,
        "ClassDeclaration": true,
        "ArrowFunctionExpression": false
      }
    }]
  }
}
```

**Python doctest**:

```python
# src/services/user_service.py
def calculate_discount(price: float, discount_percent: float) -> float:
    """
    Calculate the discounted price.

    >>> calculate_discount(100, 10)
    90.0
    >>> calculate_discount(50, 20)
    40.0
    >>> calculate_discount(0, 10)
    0.0
    """
    return price * (1 - discount_percent / 100)

# Run doctests
# python -m doctest src/services/user_service.py -v
# or pytest --doctest-modules
```

### Cuándo usarlo

Para APIs públicas y librerías. Para código interno, la documentación es opcional pero
los doctests son valiosos porque garantizan que los ejemplos funcionan.

---

## 4.8 Análisis de churn / hotspots

### Qué es

Cruza complejidad × frecuencia de cambio. Los archivos que cambian mucho Y son complejos
son candidatos a refactorización.

### Script

```bash
#!/bin/bash
# scripts/hotspots.sh
echo "=== Top 20 archivos más cambiados ==="
git log --format=format: --name-only | sort | uniq -c | sort -n | tail -20

echo ""
echo "=== Archivos más complejos (líneas) ==="
find src/ -name '*.ts' -exec wc -l {} + | sort -n | tail -20

echo ""
echo "=== Hotspots (churn × complejidad) ==="
# Combina churn con complejidad para identificar refactor candidates
for file in $(git log --format=format: --name-only | sort | uniq -c | sort -n | tail -20 | awk '{print $2}'); do
  if [ -f "$file" ]; then
    lines=$(wc -l < "$file")
    changes=$(git log --oneline -- "$file" | wc -l)
    score=$((lines * changes))
    echo "$score $changes $lines $file"
  fi
done | sort -n | tail -10
```

### Cuándo usarlo

Mensual o trimestral. Los hotspots guían la deuda técnica: refactoriza primero los
archivos que más cambian y son más complejos — es donde los bugs se esconden.

---

## Skills y GitHub Projects

### BMad Skills relevantes

- **`bmad-editorial-review-structure`** — revisión estructural del código. Útil para
  identificar módulos mal organizados, capas filtradas, y dependencias circulares que
  dependency-cruiser no detecta (ej: acoplamiento conceptual sin import directo).
- **`bmad-review-adversarial-general`** — revisión cínica de la arquitectura. Útil para
  encontrar violaciones sutiles de capas que las herramientas automatizadas no detectan.

### GitHub Projects y npm packages

| Tool | Repo / Package | Instalación |
|------|----------------|-------------|
| dependency-cruiser | [sverweij/dependency-cruiser](https://github.com/sverweij/dependency-cruiser) | `npm i -D dependency-cruiser` |
| import-linter (Python) | [seddonym/import-linter](https://github.com/seddonym/import-linter) | `pip install import-linter` |
| ArchUnit (Java) | [TNG/ArchUnit](https://github.com/TNG/ArchUnit) | Maven/Gradle |
| Semgrep | [semgrep/semgrep](https://github.com/semgrep/semgrep) | `brew install semgrep` o `pip install semgrep` |
| bandit (Python) | [PyCQA/bandit](https://github.com/PyCQA/bandit) | `pip install bandit` |
| radon (Python) | [rubik/radon](https://github.com/rubik/radon) | `pip install radon` |
| openapi-diff | [OpenAPITools/openapi-diff](https://github.com/OpenAPITools/openapi-diff) | Descargar JAR |
| API Extractor | [microsoft/rushstack](https://github.com/microsoft/rushstack) | `npm i -D @microsoft/api-extractor` |
| eslint-plugin-security | [eslint-community/eslint-plugin-security](https://github.com/eslint-community/eslint-plugin-security) | `npm i -D eslint-plugin-security` |
