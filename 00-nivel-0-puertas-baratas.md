# Nivel 0 — Puertas Baratas (Cheap Pre-Commit Gates)

> **Filosofía**: El nivel más bajo de la pirámide de testing de Uncle Bob no son tests.
> Son **puertas** — verificaciones deterministas que corren en segundos, antes de cada commit,
> y que evitan que código roto, mal formateado, o con secretos llegue al repositorio.
>
> **Objetivo**: < 10 segundos de ejecución total en el pre-commit hook.
>
> **Stack genérico**: Node.js / TypeScript / Python. Los ejemplos usan Node.js + TypeScript
> como referencia, pero los conceptos aplican a cualquier lenguaje.

---

## Tabla de Contenidos

| Gate | Tool | Tiempo objetivo |
|------|------|-----------------|
| [0.1 Formateo determinista](#01-formateo-determinista) | Prettier | < 2s |
| [0.2 Lint](#02-lint) | ESLint | < 3s |
| [0.3 Type checking estricto](#03-type-checking-estricto) | tsc / mypy | < 3s |
| [0.4 Secret scanning](#04-secret-scanning) | gitleaks | < 1s |
| [0.5 Conventional commits](#05-conventional-commits) | commitlint | < 1s |
| [0.6 Detección de código muerto](#06-detección-de-código-muerto) | knip | < 2s |
| [0.7 Detección de duplicación](#07-detección-de-duplicación) | jscpd | < 2s |
| [Pre-commit hook](#pre-commit-hook) | husky / lefthook | < 10s total |

---

## 0.1 Formateo Determinista

### Qué es

Prettier reformatea el código fuente de manera **determinista** — el mismo input siempre
produce el mismo output, sin importar quién lo ejecute ni en qué máquina. Elimina los
debates sobre estilo en code reviews: si Prettier lo generó, pasa.

### Por qué importa

Uncle Bob defiende que el código debe verse como si lo hubiera escrito **una sola persona**.
El formateo determinista es la base de esa consistencia. En *"Clean Code"* argumenta que
el estilo no es cosmético — es parte de la legibilidad, y la legibilidad es parte de
la mantenibilidad. Si cada desarrollador formatea distinto, el diff se llena de ruido
visual que oculta cambios reales.

> *"El código limpio siempre parece escrito por alguien a quien le importa."* — Robert C. Martin

El formateo automático también reduce la **carga cognitiva** del code review: el revisor
puede ignorar el formato y concentrarse en la lógica.

### Stack

- **Node.js / TypeScript**: Prettier
- **Python**: Black + isort
- **Rust**: rustfmt
- **Go**: `gofmt` (integrado en el toolchain)

### Cómo implementarlo

```bash
# Node.js / TypeScript
npm install -D prettier

# Python
pip install black isort
```

Crear el archivo `.prettierrc` en la raíz del proyecto:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2
}
```

Crear `.prettierignore` para excluir archivos generados:

```text
# .prettierignore
node_modules/
dist/
build/
*.min.js
package-lock.json
yarn.lock
pnpm-lock.yaml
```

Ejecutar manualmente para verificar:

```bash
# Ver qué archivos están mal formateados (sin modificar)
npx prettier --check .

# Formatear todo
npx prettier --write .
```

### Ejemplo de código

**`.prettierrc`**:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2
}
```

**Script en `package.json`**:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

**Python equivalente** (`pyproject.toml`):

```toml
[tool.black]
line-length = 100
target-version = ['py311']

[tool.isort]
profile = "black"
line_length = 100
```

### Cuándo usarlo

Siempre. No hay proyecto demasiado pequeño para formateo determinista. Es la puerta más
barata y la de mayor retorno: cuesta < 2s y elimina ruido de diffs para siempre.

---

## 0.2 Lint

### Qué es

ESLint analiza el código estáticamente para detectar **patrones problemáticos**: variables
sin usar, tipos `any` implícitos, `console.log` olvidados en producción, mutaciones
inesperadas. A diferencia de Prettier (que solo reformatea), ESLint encuentra **bugs**
antes de que se ejecuten.

### Por qué importa

Uncle Bob distingue entre **errores de compilación** (el compilador los atrapa) y
**errores de diseño** (el linter los atrapa). En *"Clean Code"*, el capítulo sobre
"Reglas de formato" y el capítulo sobre "Errores" convergen en una idea: **cuanto antes
detectes un error, más barato es arreglarlo**.

El linter es la primera línea de defensa contra:
- Código muerto (variables sin usar)
- Abstracciones filtradas (tipos `any` que rompen el sistema de tipos)
- Debugging residual (`console.log` en producción)
- Mutaciones accidentales (`let` donde debería ser `const`)

> *"Dejar código muerto es como dejar basura en la sala."* — adaptado de Robert C. Martin

### Stack

- **Node.js / TypeScript**: ESLint + `@typescript-eslint`
- **Python**: Ruff o Pylint + flake8
- **Rust**: Clippy
- **Go**: `golangci-lint`

### Cómo implementarlo

```bash
# Node.js / TypeScript
npm install -D eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser

# Python
pip install ruff
```

### Ejemplo de código

**`.eslintrc.json`** (Node.js / TypeScript):

```json
{
  "root": true,
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "ecmaVersion": 2024,
    "sourceType": "module",
    "project": "./tsconfig.json"
  },
  "plugins": ["@typescript-eslint"],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-requiring-type-checking"
  ],
  "rules": {
    "no-unused-vars": "off",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/no-explicit-any": "error",
    "prefer-const": "error",
    "no-console": ["error", { "allow": ["warn", "error"] }]
  },
  "overrides": [
    {
      "files": ["src/server.ts", "src/index.ts"],
      "rules": {
        "no-console": "off"
      }
    }
  ],
  "ignorePatterns": ["dist/", "node_modules/"]
}
```

**Script en `package.json`**:

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix"
  }
}
```

**Python equivalente** (`pyproject.toml`):

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "W", "I", "N", "B", "UP", "SIM"]
ignore = ["E501"]
```

### Cuándo usarlo

Siempre. El linter debe correr en pre-commit y en el quality gate. Configúralo con
`--max-warnings 0` en CI para que los warnings también bloqueen.

---

## 0.3 Type Checking Estricto

### Qué es

`tsc --noEmit` (TypeScript) o `mypy --strict` (Python) ejecuta el compilador/type-checker
**sin generar archivos de salida** — solo verifica que los tipos sean correctos. Es la
verificación más completa de que el código es internamente consistente a nivel de tipos.

### Por qué importa

El sistema de tipos es un **sistema de pruebas automáticas** que vive en el compilador.
Cada tipo es una aserción sobre el código. Uncle Bob lo compara con el "gauntlet" — el
túnel de restricciones que el código debe superar antes de ser considerado válido.

TypeScript en modo `strict` activa:
- `strictNullChecks` — `null` y `undefined` no son asignables a otros tipos
- `noImplicitAny` — prohíbe `any` implícito
- `strictFunctionTypes` — contravarianza correcta en parámetros de funciones
- `noUnusedLocals` / `noUnusedParameters` — prohíbe variables sin usar
- `exactOptionalPropertyTypes` — distingue `undefined` de "ausente"

### Stack

- **TypeScript**: `tsc --noEmit` con `strict: true`
- **Python**: `mypy --strict` o `pyright --strict`
- **Rust**: el compilador (`cargo check`) es inherentemente estricto

### Cómo implementarlo

```bash
# TypeScript
npx tsc --noEmit

# Python
mypy --strict src/
```

### Ejemplo de código

**`tsconfig.json`**:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noEmit": true,
    "skipLibCheck": true,
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Script en `package.json`**:

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit"
  }
}
```

### Cuándo usarlo

Siempre. El type checking debe ser la primera puerta del pre-commit hook. Si `tsc` falla,
no hay razón para correr nada más. Para Python, `mypy --strict` es la puerta equivalente.

---

## 0.4 Secret Scanning

### Qué es

[gitleaks](https://github.com/gitleaks/gitleaks) escanea el código y el historial de git
en busca de **secretos hardcodeados**: API keys, tokens, contraseñas, claves privadas.

### Por qué importa

Un secreto commiteado al repositorio es una vulnerabilidad **permanente** — incluso si lo
eliminas en el siguiente commit, sigue en el historial de git. gitleaks detecta patrones
conocidos (AWS keys, JWTs, claves privadas RSA) y patrones custom que definas.

### Stack

- **Cualquier lenguaje**: gitleaks (Go binary, sin dependencias)
- **Alternativa**: [Trivy](https://github.com/aquasecurity/trivy) (también escanea contenedores)

### Cómo implementarlo

```bash
# Instalar
brew install gitleaks
# o
pip install gitleaks
# o descargar el binary de GitHub releases

# Escanear el repositorio
gitleaks detect --source . --no-banner

# Escanear un commit específico
gitleaks detect --source . --log-opts="HEAD~1..HEAD"
```

### Ejemplo de código

**`.gitleaks.toml`** (reglas custom):

```toml
title = "Custom Secret Rules"

[[rules]]
id = "custom-api-key"
description = "Custom API key pattern"
regex = '''API_KEY\s*=\s*["'][A-Za-z0-9]{32,}["']'''
tags = ["api-key"]

[[rules]]
id = "jwt-token"
description = "JWT token"
regex = '''eyJ[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+\.[A-Za-z0-9-_]*'''
tags = ["jwt"]

[allowlist]
description = "Allowlisted false positives"
paths = [
  '''^vendor/''',
  '''^node_modules/''',
  '''\.example$''',
]
```

### Cuándo usarlo

Siempre. gitleaks debe correr en pre-commit (escanea solo el diff) y en el quality gate
(escanea todo el repo). Es la puerta más barata contra la vulnerabilidad más cara.

---

## 0.5 Conventional Commits

### Qué es

[commitlint](https://github.com/conventional-changelog/commitlint) verifica que los
mensajes de commit sigan el formato [Conventional Commits](https://www.conventionalcommits.org/):
`type(scope): description`.

### Por qué importa

Los commits son documentación. Un historial de commits con formato consistente permite:
- Generar changelogs automáticamente
- Determinar si un commit es breaking (para semver)
- Filtrar commits por tipo (`feat`, `fix`, `refactor`, `test`, `docs`)
- Facilitar revert y bisect

### Stack

- **Cualquier lenguaje**: commitlint + `@commitlint/config-conventional`

### Cómo implementarlo

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
```

### Ejemplo de código

**`commitlint.config.js`**:

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // nueva funcionalidad
        'fix',      // bug fix
        'docs',     // solo documentación
        'style',    // formato, sin cambio de código
        'refactor', // refactor sin cambio de comportamiento
        'test',     // agregar o corregir tests
        'chore',    // build, deps, config
        'ci',       // CI/CD
        'perf',     // mejora de rendimiento
        'revert',   // revert de un commit
      ],
    ],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [1, 'always', 100],
  },
};
```

**Ejemplos de commits válidos**:

```text
feat(orders): add order cancellation endpoint
fix(auth): handle expired JWT token gracefully
test(user-service): add property-based tests for email validation
docs(readme): update installation instructions
refactor(payment): extract payment processor strategy
```

### Cuándo usarlo

Siempre que el proyecto tenga más de un desarrollador (humano o agente de IA). Para
proyectos de un solo desarrollador, es opcional pero recomendado para mantener disciplina.

---

## 0.6 Detección de Código Muerto

### Qué es

[knip](https://github.com/webpro-nl/knip) detecta **código muerto** en proyectos
TypeScript/JavaScript: exports sin usar, archivos no importados, dependencias no
utilizadas, tipos sin referenciar.

### Por qué importa

El código muerto es deuda técnica invisible. Un agente de IA que escribe código produce
exports, funciones y tipos que nadie usa. Sin knip, estos se acumulan indefinidamente.

> *"Dejar código muerto es como dejar basura en la sala."* — adaptado de Robert C. Martin

### Stack

- **Node.js / TypeScript**: [knip](https://github.com/webpro-nl/knip)
- **Python**: [vulture](https://github.com/jendrikseipp/vulture) o [deptry](https://github.com/fpgmaas/deptry)
- **Rust**: `cargo udeps` o warnings del compilador (`unused`)

### Cómo implementarlo

```bash
# Node.js / TypeScript
npm install -D knip
npx knip

# Python
pip install vulture
vulture src/
```

### Ejemplo de código

**`knip.json`**:

```json
{
  "entry": ["src/index.ts", "src/server.ts"],
  "project": ["src/**/*.ts"],
  "ignore": ["src/**/*.test.ts", "src/**/*.spec.ts"],
  "ignoreBinaries": ["docker-compose"],
  "ignoreDependencies": ["@types/node"]
}
```

**Script en `package.json`**:

```json
{
  "scripts": {
    "knip": "knip"
  }
}
```

### Cuándo usarlo

En el quality gate (pre-push o PR). No es necesario en cada commit porque puede ser lento
en proyectos grandes. Para Python, `vulture` tiene falsos positivos — combínalo con
`deptry` para dependencias no usadas.

---

## 0.7 Detección de Duplicación

### Qué es

[jscpd](https://github.com/kucherenko/jscpd) detecta **bloques de código duplicado**
(copy-paste) en el codebase. Reporta el porcentaje de duplicación y los bloques
específicos.

### Por qué importa

La duplicación es el enemigo de la mantenibilidad. Si un bug se arregla en una copia pero
no en la otra, el bug persiste. Uncle Bob lo llama el "principio DRY" (Don't Repeat
Yourself) — cada pieza de conocimiento debe tener una representación única en el sistema.

### Stack

- **Cualquier lenguaje**: [jscpd](https://github.com/kucherenko/jscpd) (soporta 150+ lenguajes)
- **Python**: [PMD CPD](https://pmd.github.io/) o jscpd

### Cómo implementarlo

```bash
npm install -D jscpd
npx jscpd src/ --threshold 5
```

### Ejemplo de código

**`.jscpd.json`**:

```json
{
  "threshold": 5,
  "reporters": ["html", "console", "json"],
  "ignore": ["**/*.test.ts", "**/*.spec.ts", "**/node_modules/**", "**/dist/**"],
  "format": ["typescript", "javascript", "python"],
  "min-lines": 5,
  "min-tokens": 50
}
```

**Script en `package.json`**:

```json
{
  "scripts": {
    "jscpd": "jscpd src/ --threshold 5"
  }
}
```

### Cuándo usarlo

En el quality gate semanal o nightly. La duplicación no bloquea commits pero debe
monitorearse para evitar degradación gradual. Umbral recomendado: < 5%.

---

## Pre-commit Hook

### Qué es

[husky](https://github.com/typicode/husky) o [lefthook](https://github.com/evilmartians/lefthook)
configura hooks de git que ejecutan las puertas automáticamente antes de cada commit.

### Cómo implementarlo

```bash
# Con husky
npm install -D husky
npx husky init

# Con lefthook (alternativa más rápida)
# ver: https://github.com/evilmartians/lefthook
```

### Ejemplo de código

**`.husky/pre-commit`**:

```bash
#!/bin/sh
echo "=== Pre-commit gates ==="

echo "1. Prettier..."
npx prettier --check . || { echo "❌ Format check failed. Run: npm run format"; exit 1; }

echo "2. ESLint..."
npx eslint . --ext .ts,.tsx --max-warnings 0 || { echo "❌ Lint failed"; exit 1; }

echo "3. TypeScript..."
npx tsc --noEmit || { echo "❌ Type check failed"; exit 1; }

echo "4. gitleaks..."
gitleaks detect --source . --no-banner --log-opts="HEAD" || { echo "❌ Secrets detected"; exit 1; }

echo "=== All pre-commit gates passed ==="
```

**`.husky/commit-msg`** (para commitlint):

```bash
#!/bin/sh
npx --no-install commitlint --edit "$1"
```

### Cuándo usarlo

Siempre. El pre-commit hook es el mecanismo que hace que las puertas sean automáticas.
Sin hook, las puertas son manuales y eventualmente se saltan.

---

## Skills y GitHub Projects

### BMad Skills relevantes

- **`bmad-editorial-review-structure`** — revisión estructural de la configuración de
  puertas. Útil para auditar que las reglas de ESLint/Prettier sean coherentes y no
  redundantes. [Ver skill](https://github.com/cognition-ai/bmad)
- **`bmad-review-adversarial-general`** — revisión cínica de la configuración de puertas.
  Útil para encontrar gaps: "¿qué pasa si el desarrollador hace `--no-verify`? ¿hay
  otra capa que lo detecte?"

### GitHub Projects y npm packages

| Tool | Repo / Package | Instalación |
|------|----------------|-------------|
| Prettier | [prettier/prettier](https://github.com/prettier/prettier) | `npm i -D prettier` |
| ESLint | [eslint/eslint](https://github.com/eslint/eslint) | `npm i -D eslint` |
| TypeScript ESLint | [typescript-eslint](https://github.com/typescript-eslint/typescript-eslint) | `npm i -D @typescript-eslint/eslint-plugin @typescript-eslint/parser` |
| gitleaks | [gitleaks/gitleaks](https://github.com/gitleaks/gitleaks) | `brew install gitleaks` |
| commitlint | [conventional-changelog/commitlint](https://github.com/conventional-changelog/commitlint) | `npm i -D @commitlint/cli @commitlint/config-conventional` |
| knip | [webpro-nl/knip](https://github.com/webpro-nl/knip) | `npm i -D knip` |
| jscpd | [kucherenko/jscpd](https://github.com/kucherenko/jscpd) | `npm i -D jscpd` |
| husky | [typicode/husky](https://github.com/typicode/husky) | `npm i -D husky` |
| lefthook | [evilmartians/lefthook](https://github.com/evilmartians/lefthook) | `brew install lefthook` |
| Black (Python) | [psf/black](https://github.com/psf/black) | `pip install black` |
| Ruff (Python) | [astral-sh/ruff](https://github.com/astral-sh/ruff) | `pip install ruff` |
| mypy (Python) | [python/mypy](https://github.com/python/mypy) | `pip install mypy` |
| vulture (Python) | [jendrikseipp/vulture](https://github.com/jendrikseipp/vulture) | `pip install vulture` |
