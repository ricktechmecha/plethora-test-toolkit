# Test Plethora Toolkit — Guía Universal de Testing

> **Filosofía: la "plétora" de Uncle Bob**
>
> Robert C. Martin (Uncle Bob) propone una estrategia de testing basada en
> **restricciones, no en lectura del código**. La idea central es:
>
> 1. **No leas el código que tus agentes escriben.** Rodealo de
>    restricciones (tests) que describan el comportamiento esperado.
> 2. **La plétora de tests** — un conjunto exhaustivo y multi-nivel que
>    cubre desde la unidad más pequeña hasta la aceptación end-to-end,
>    pasando por mutación, carga, seguridad, propiedades y gobernanza.
> 3. **Cada restricción es una decisión de diseño diferida.** Si un test
>    falla, el código bajo prueba no cumple el contrato; no se necesita
>    inspeccionar la implementación para saber *qué* falló.
> 4. **Cobertura sin obsesión.** La cobertura es un termómetro, no un fin.
>    Lo que importa es que las restricciones capturen intención.
>
> Este toolkit es una **referencia reutilizable** para cualquier proyecto.
> Documenta cada tipo de test que puedes implementar, cómo configurarlo
> desde cero, y qué herramientas / proyectos / skills existen para cada
> nivel. No está atado a ningún proyecto, stack ni dominio específico.

---

## Cómo usar este toolkit

Copia los `.md` que apliquen a tu proyecto. Cada `.md` es independiente y
explica cómo implementar ese tipo de test desde cero.

1. **Lee este README** para entender el panorama completo de niveles.
2. **Abre el archivo del nivel** que necesitas implementar
   (`00-nivel-0-*.md`, `01-nivel-1-*.md`, etc.).
3. **Copia la configuración** y adáptala al stack de tu proyecto.
4. **Usa el inventario** (`templates/INVENTARIO.template.md`) como plantilla
   para documentar lo que ya tienes.
5. **Usa el plan** (`templates/MATRIZ_TRAZABILIDAD.template.md`) como plantilla para
   priorizar lo que falta.

> **Funciona con cualquier stack:** Node.js, TypeScript, Python, Go, Rust,
> Java, etc. Las herramientas listadas por nivel tienen equivalentes en
> cada lenguaje.

---

## Tabla de Contenidos

| # | Nivel | Archivo | Descripción |
|---|-------|---------|-------------|
| 0 | Puertas Baratas (Pre-commit Gates) | [`00-nivel-0-puertas-baratas.md`](./00-nivel-0-puertas-baratas.md) | Formateo, lint, type-check, secret scanning, conventional commits, dead code, duplicación |
| 1 | Correctitud Funcional | [`01-nivel-1-correctitud-funcional.md`](./01-nivel-1-correctitud-funcional.md) | Unit tests, integration tests, E2E, contract tests, BDD |
| 2 | Generación Automática de Casos | [`02-nivel-2-generacion-automatica.md`](./02-nivel-2-generacion-automatica.md) | Property-based testing, fuzzing, schema fuzzing, metamorphic testing |
| 3 | Testear los Tests | [`03-nivel-3-testear-los-tests.md`](./03-nivel-3-testear-los-tests.md) | Mutation testing, patch coverage, test effectiveness |
| 4 | Estructura y Arquitectura | [`04-nivel-4-estructura-arquitectura.md`](./04-nivel-4-estructura-arquitectura.md) | Dependency rules, cycle detection, layer enforcement, SAST |
| 5 | No Funcional | [`05-nivel-5-no-funcional.md`](./05-nivel-5-no-funcional.md) | Performance, load, soak, accessibility, visual regression, observability |
| 6 | Seguridad y Privacidad | [`06-nivel-6-seguridad-privacidad.md`](./06-nivel-6-seguridad-privacidad.md) | SAST, DAST, SCA, SBOM, secret scanning profundo, threat modeling |
| 7 | Datos e IA | [`07-nivel-7-datos-ia.md`](./07-nivel-7-datos-ia.md) | LLM evals, prompt testing, data validation, AI contract tests |
| 8 | Proceso y Gobierno | [`08-nivel-8-proceso-gobierno.md`](./08-nivel-8-proceso-gobierno.md) | Traceability, ADRs, spec review, adversarial review, progressive delivery |
| — | Inventario | [`templates/INVENTARIO.template.md`](./templates/INVENTARIO.template.md) | Plantilla para documentar tests existentes |
| — | Trazabilidad | [`templates/MATRIZ_TRAZABILIDAD.template.md`](./templates/MATRIZ_TRAZABILIDAD.template.md) | Plantilla requisito → test |
| — | Caso de estudio | [`case-study/CASE_STUDY.md`](./case-study/CASE_STUDY.md) | Aplicación real en un SaaS multi-tenant en producción |

---

## Niveles Detallados

### Nivel 0 — Puertas Baratas (Cheap Pre-Commit Gates)

> El nivel más bajo no son tests. Son **puertas** — verificaciones
> deterministas que corren en segundos, antes de cada commit, y que
> evitan que código roto, mal formateado, o con secretos llegue al
> repositorio. **Objetivo: < 10 segundos.**

**Qué incluye:** Formateo determinista (Prettier), lint (ESLint),
type-check estricto (tsc/mypy/pyright), secret scanning (gitleaks),
conventional commits (commitlint), detección de código muerto (knip),
detección de duplicación (jscpd), pre-commit hooks (husky + lint-staged).

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-review-adversarial-general` | Review cínico/adversarial — útil para revisar que las puertas del Nivel 0 realmente bloquean lo que deben bloquear |
| `bmad-editorial-review-structure` | Review estructural — verifica que la configuración de puertas sea consistente y completa |

**GitHub Projects / Tools:**

| Tool | Lenguaje | Descripción | Repo |
|------|----------|-------------|------|
| [husky](https://github.com/typicode/husky) | JS/TS | Git hooks made easy | `typicode/husky` |
| [lint-staged](https://github.com/lint-staged/lint-staged) | JS/TS | Run linters on staged files only | `lint-staged/lint-staged` |
| [Prettier](https://github.com/prettier/prettier) | Multi | Opinionated code formatter | `prettier/prettier` |
| [ESLint](https://github.com/eslint/eslint) | JS/TS | Pluggable linter | `eslint/eslint` |
| [gitleaks](https://github.com/gitleaks/gitleaks) | Multi (Go binary) | Detect hardcoded secrets in git history | `gitleaks/gitleaks` |
| [knip](https://github.com/webpro-nl/knip) | JS/TS | Find unused files, exports, dependencies | `webpro-nl/knip` |
| [jscpd](https://github.com/kucherenko/jscpd) | Multi | Copy-paste detector for source code | `kucherenko/jscpd` |
| [commitlint](https://github.com/conventional-changelog/commitlint) | JS/TS | Lint commit messages per conventional commits | `conventional-changelog/commitlint` |

> **Equivalentes otros lenguajes:** Python → `ruff`, `black`, `mypy`,
> `pre-commit`; Go → `gofmt`, `golangci-lint`, `pre-commit`; Rust →
> `rustfmt`, `clippy`.

---

### Nivel 1 — Correctitud Funcional (El Núcleo)

> "El código tuvo que pasar el gauntlet completo de restricciones." — Uncle Bob

Este es el corazón del harness. Sin estos tests, no hay nada. Un agente
de IA que escribe código sin tests funcionales es un caos productivo.

**Qué incluye:** Unit tests, integration tests, E2E (HTTP y browser),
contract testing (consumer-driven), BDD (Gherkin/Cucumber), snapshot
testing.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-qa-generate-e2e-tests` | Genera tests E2E automatizados para features existentes — directamente aplicable al Nivel 1 |
| `bmad-testarch-trace` | Genera matriz de trazabilidad — mapea cada requisito a sus tests funcionales |
| `bmad-review-adversarial-general` | Review adversarial de los tests — ¿realmente prueban el comportamiento o sólo la implementación? |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [Jest](https://github.com/jestjs/jest) | Unit/Integration (JS/TS) | Delightful testing framework | `jestjs/jest` |
| [Playwright](https://github.com/microsoft/playwright) | E2E (browser) | Reliable end-to-end testing for web apps | `microsoft/playwright` |
| [Cypress](https://github.com/cypress-io/cypress) | E2E (browser) | Fast, easy testing for anything in a browser | `cypress-io/cypress` |
| [Pact](https://github.com/pact-foundation/pact-js) | Contract | Consumer-driven contract testing | `pact-foundation/pact-js` |
| [jest-cucumber](https://github.com/tony-kerz/jest-cucumber) | BDD (JS/TS) | Cucumber-style BDD with Jest | `tony-kerz/jest-cucumber` |
| [Cucumber](https://github.com/cucumber/cucumber-js) | BDD (Multi) | BDD with Gherkin scenarios | `cucumber/cucumber-js` |

> **Equivalentes otros lenguajes:** Python → `pytest`, `behave`, `pact-python`;
> Go → `testing`, `testify`, `pact-go`; Java → `JUnit 5`, `Cucumber-JVM`.

---

### Nivel 2 — Generación Automática de Casos

> "El agente escribe la propiedad, la máquina genera los miles de casos." — Uncle Bob

Este es el **verdadero apalancamiento** con agentes de IA. En lugar de
escribir 100 tests manuales, el agente escribe 1 propiedad y la máquina
genera 10,000 casos.

**Qué incluye:** Property-based testing, fuzzing, schema fuzzing,
metamorphic testing, differential testing.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-review-adversarial-general` | Review adversarial — verifica que las propiedades sean significativas y no triviales |
| `bmad-eval-runner` | Ejecuta evals en entorno limpio — útil para validar que los casos generados son deterministas |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [fast-check](https://github.com/dubzzz/fast-check) | Property-based (JS/TS) | Property-based testing framework | `dubzzz/fast-check` |
| [Hypothesis](https://github.com/HypothesisWorks/hypothesis) | Property-based (Python) | Property-based testing for Python | `HypothesisWorks/hypothesis` |
| [Schemathesis](https://github.com/schemathesis/schemathesis) | Schema fuzzing (Python) | Property-based testing for OpenAPI/GraphQL | `schemathesis/schemathesis` |
| [AFL++](https://github.com/AFLplusplus/AFLplusplus) | Fuzzing (C/C++/multi) | Advanced fuzzer for coverage-guided fuzzing | `AFLplusplus/AFLplusplus` |
| [libFuzzer](https://github.com/llvm/llvm-project) | Fuzzing (C/C++) | In-process fuzzer part of LLVM | `llvm/llvm-project` |

> **Equivalentes otros lenguajes:** Go → `go fuzz` (built-in), `gopter`;
> Rust → `proptest`, `cargo-fuzz`; Java → `jqwik`.

---

### Nivel 3 — Testear los Tests (No Negociable)

> "Si el mismo agente escribe el código y su prueba, la prueba tiende a codificar el bug." — Uncle Bob

**Este nivel es NO NEGOCIABLE** en un harness con agentes de IA. Sin
esto, todo lo demás es teatro.

**Qué incluye:** Mutation testing, patch coverage, test effectiveness
analysis, coverage thresholds como piso.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-eval-runner` | Ejecuta evals en entorno limpio — ideal para correr mutation testing en aislamiento |
| `bmad-review-adversarial-general` | Review adversarial — cuestiona si los tests sobreviven a mutaciones reales |

**GitHub Projects / Tools:**

| Tool | Lenguaje | Descripción | Repo |
|------|----------|-------------|------|
| [Stryker](https://github.com/stryker-mutator/stryker-js) | JS/TS | Mutation testing framework | `stryker-mutator/stryker-js` |
| [mutmut](https://github.com/boxed/mutmut) | Python | Mutation testing for Python | `boxed/mutmut` |
| [Cosmic Ray](https://github.com/sixty-north/cosmic-ray) | Python | Mutation testing tool | `sixty-north/cosmic-ray` |
| [PIT](https://github.com/hcoles/pitest) | Java | Mutation testing system | `hcoles/pitest` |

> **Equivalentes otros lenguajes:** Go → `go-mutesting`; Rust → `cargo-mutants`;
> C# → `Stryker.NET`.

---

### Nivel 4 — Estructura y Arquitectura

> "Las métricas no capturan vulnerabilidades ni código muerto que un ingeniero reconoce de vista." — Grady Booch

Métricas de estructura, acoplamiento y complejidad. Codifican las reglas
de capas y prohíben ciclos.

**Qué incluye:** Dependency rules, cycle detection, layer enforcement,
SAST (static analysis), architectural fitness functions.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-editorial-review-structure` | Review estructural — verifica que las reglas de arquitectura estén codificadas como tests |
| `bmad-review-adversarial-general` | Review adversarial — cuestiona si las reglas de arquitectura realmente previenen violaciones |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [dependency-cruiser](https://github.com/sverweij/dependency-cruiser) | JS/TS | Validate and visualize dependencies | `sverweij/dependency-cruiser` |
| [Semgrep](https://github.com/semgrep/semgrep) | Multi (SAST) | Static analysis at ludicrous speed | `semgrep/semgrep` |
| [SonarQube](https://github.com/SonarSource/sonarqube) | Multi | Continuous code quality & security | `SonarSource/sonarqube` |
| [ArchUnit](https://github.com/TNG/ArchUnit) | Java | Test your architecture | `TNG/ArchUnit` |

> **Equivalentes otros lenguajes:** Python → `import-linter`, `pylint`;
> Go → `go-arch-lint`, `depguard`; C# → `NetArchTest`; TS/JS →
> `projectlint`, `madge`.

---

### Nivel 5 — No Funcional

Tests de rendimiento, resiliencia, observabilidad, accesibilidad, visual
regression, DR drills.

**Qué incluye:** Load testing, soak testing, spike testing, performance
profiling, accessibility testing (a11y), visual regression testing,
observability tests, disaster recovery drills.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-qa-generate-e2e-tests` | Genera tests E2E — extensible para generar tests de accesibilidad y visual regression |
| `bmad-review-adversarial-general` | Review adversarial — cuestiona si los umbrales de performance son realistas |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [Artillery](https://github.com/artilleryio/artillery) | Load (JS) | Cloud-scale load testing | `artilleryio/artillery` |
| [k6](https://github.com/grafana/k6) | Load (Go/JS) | Modern load testing tool | `grafana/k6` |
| [Locust](https://github.com/locustio/locust) | Load (Python) | Scalable user load testing | `locustio/locust` |
| [Gatling](https://github.com/gatling/gatling) | Load (Scala) | Load testing framework | `gatling/gatling` |
| [clinic.js](https://github.com/clinicjs/node-clinic) | Profiling (Node.js) | Diagnose performance issues | `clinicjs/node-clinic` |
| [axe-core](https://github.com/dequelabs/axe-core) | Accessibility | Accessibility testing engine | `dequelabs/axe-core` |
| [Percy](https://github.com/percy/percy-playwright-js) | Visual regression | Visual review and testing | `percy/percy-playwright-js` |

> **Equivalentes otros lenguajes:** Python profiling → `py-spy`, `cProfile`;
> Go → `pprof`; Rust → `flamegraph`.

---

### Nivel 6 — Seguridad y Privacidad

Seguridad no es opcional. Si tu sistema maneja datos de usuarios, datos
sensibles, o expone APIs, este nivel es obligatorio.

**Qué incluye:** SAST (static analysis), DAST (dynamic analysis), SCA
(software composition analysis), SBOM (software bill of materials),
secret scanning profundo, threat modeling, container scanning.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-review-adversarial-general` | Review adversarial — actúa como red team básico, cuestiona supuestos de seguridad |
| `bmad-testarch-trace` | Trazabilidad — mapea requisitos de seguridad a tests que los validan |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [OWASP ZAP](https://github.com/zaproxy/zaproxy) | DAST | Web app security scanner | `zaproxy/zaproxy` |
| [Nuclei](https://github.com/projectdiscovery/nuclei) | DAST | Fast, customizable vulnerability scanner | `projectdiscovery/nuclei` |
| [Trivy](https://github.com/aquasecurity/trivy) | Container/SCA | Comprehensive security scanner | `aquasecurity/trivy` |
| [osv-scanner](https://github.com/google/osv-scanner) | SCA | Vulnerability scanner based on OSV database | `google/osv-scanner` |
| [Syft](https://github.com/anchore/syft) | SBOM | Generate SBOM from container images | `anchore/syft` |
| [Checkov](https://github.com/bridgecrewio/checkov) | IaC security | Scan cloud infrastructure for misconfigurations | `bridgecrewio/checkov` |

> **Equivalentes:** Python → `bandit`, `safety`; Go → `gosec`; Java →
> `SpotBugs`; Docker → `docker-scout`, `grype`.

---

### Nivel 7 — Datos e IA

Si tu sistema usa LLMs, OCR, clasificación automática, o procesa datos
sensibles con pipelines de IA, este nivel no es opcional.

**Qué incluye:** LLM evals, prompt testing, AI contract tests, data
validation, hallucination detection, tool-calling verification, RAG
quality metrics.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-eval-runner` | Ejecuta evals en entorno limpio — diseñado específicamente para correr eval sets de IA de forma determinista |
| `bmad-review-adversarial-general` | Review adversarial — cuestiona si los evals de IA realmente capturan casos de fallo LLM |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [promptfoo](https://github.com/promptfoo/promptfoo) | LLM evals | Test & evaluate LLM prompts and RAG | `promptfoo/promptfoo` |
| [Langfuse](https://github.com/langfuse/langfuse) | LLM observability | Open-source LLM engineering platform | `langfuse/langfuse` |
| [Helicone](https://github.com/Helicone/helicone) | LLM observability | Open-source observability for LLMs | `Helicone/helicone` |
| Custom evals | LLM evals | Frameworks propios con test cases curados | — |

> **Notas:** Para data validation en pipelines ETL, ver
> [Great Expectations](https://github.com/great-expectations/great_expectations)
> (Python). Para RAG evaluation, ver
> [Ragas](https://github.com/explodinggradients/ragas).

---

### Nivel 8 — Proceso y Gobierno del Harness

> "La separación de roles es el mecanismo, no el volumen de tests." — Uncle Bob

**Qué incluye:** Traceability matrix (requisito → test), ADRs
(Architecture Decision Records), spec ambiguity review, adversarial
review, progressive delivery, post-release monitoring, governance de
cobertura.

#### Skills y GitHub Projects

**BMad Skills (ecc-enginelegal):**

| Skill | Descripción |
|-------|-------------|
| `bmad-testarch-trace` | Genera matriz de trazabilidad y decisión de quality gate — la herramienta central del Nivel 8 |
| `bmad-review-adversarial-general` | Review cínico/adversarial — el rol del "red teamer" en el proceso de gobierno |
| `bmad-editorial-review-structure` | Review estructural — verifica que la documentación de gobierno sea consistente |
| `bmad-eval-runner` | Ejecuta evals en limpio — útil para validar que el proceso de gobierno es reproducible |

**GitHub Projects / Tools:**

| Tool | Tipo | Descripción | Repo |
|------|------|-------------|------|
| [MADR](https://github.com/adr/madr) | ADR | Markdown Architecture Decision Records | `adr/madr` |
| [log4brains](https://github.com/thomvaill/log4brains) | ADR | ADR management & preview | `thomvaill/log4brains` |
| Custom traceability | Traceability | Matriz propia requisito → test (plantilla en `templates/INVENTARIO.template.md`) | — |

> **Notas:** Para traceability avanzada, ver
> [ReqIF](https://www.omg.org/spec/ReqIF/) tools y
> [ReqView](https://github.com/ReqView/reqview). Para governance de
> cobertura, integrar coverage reports con SonarQube o Codecov.

---

## Estrategia de Ejecución de 4 Tiers

La plétora no se ejecuta toda de golpe. Se divide en **4 tiers** según
frecuencia y presupuesto de tiempo:

### Tier 1 — Pre-commit (`< 10s`)

**Objetivo:** feedback instantáneo al desarrollador antes de un commit.

| Qué se ejecuta | Herramienta | Tiempo objetivo |
|----------------|-------------|-----------------|
| Formateo determinista | Prettier / black / gofmt | < 2s |
| Lint incremental | ESLint / ruff / golangci-lint | < 3s |
| Type-check (incremental) | tsc / mypy / pyright | < 3s |
| Secret scanning | gitleaks | < 1s |
| Unit tests críticos (smoke) | Jest / pytest (`--onlyChanged`) | < 4s |

**Disparador:** `git pre-commit` hook (husky / lefthook / pre-commit).

### Tier 2 — Pull Request (`< 10 min`)

**Objetivo:** validación completa antes de merge.

| Qué se ejecuta | Herramienta | Tiempo objetivo |
|----------------|-------------|-----------------|
| Suite completa de unit tests | Jest / pytest / go test | < 3 min |
| Coverage check (umbrales) | Istanbul / coverage.py | incluido |
| E2E backend (HTTP) | Jest+supertest / pytest / httptest | < 2 min |
| E2E frontend (smoke) | Playwright / Cypress | < 4 min |
| Mutation testing (subset crítico) | Stryker / mutmut | < 1 min |

**Disparador:** manual o hook de PR.

### Tier 3 — Nightly (`< 1h`)

**Objetivo:** validación profunda que no cabe en un PR.

| Qué se ejecuta | Herramienta | Tiempo objetivo |
|----------------|-------------|-----------------|
| E2E frontend completo | Playwright / Cypress | < 20 min |
| Mutation testing completo | Stryker / mutmut | < 30 min |
| Property-based / fuzzing | fast-check / Hypothesis | < 10 min |
| Security tests (DAST) | OWASP ZAP / Nuclei | < 10 min |
| Load testing (smoke) | Artillery / k6 | < 10 min |

**Disparador:** cron nocturno / script manual.

### Tier 4 — Semanal (`< 4h`)

**Objetivo:** validación de no-regresión a gran escala y análisis
estratégico.

| Qué se ejecuta | Herramienta | Tiempo objetivo |
|----------------|-------------|-----------------|
| Load testing completo (escenarios) | Artillery / k6 / Locust | < 45 min |
| Mutation testing full + reporte | Stryker / mutmut | < 60 min |
| SCA / dependency scanning | osv-scanner / Trivy | < 10 min |
| SBOM generation | Syft | < 5 min |
| Soak testing | k6 / Locust | < 60 min |
| DR drills | manual / automatizado | < 30 min |
| LLM eval sets (si aplica) | promptfoo / custom | < 30 min |
| Review de cobertura y gaps | manual | — |

**Disparador:** cron semanal / script manual.

---

## Los 5 Antídotos

Uncle Bob identifica **5 antídotos** contra los falsos positivos de
cobertura — técnicas que complementan los tests tradicionales cuando un
agente de IA escribe tanto el código como los tests:

| # | Antídoto | Qué hace | Nivel | Herramienta |
|---|----------|----------|-------|-------------|
| 1 | **Property-based testing** | El agente escribe la *propiedad*, la máquina genera miles de casos | 2 | fast-check, Hypothesis |
| 2 | **Metamorphic testing** | Define relaciones entre entradas/salidas que deben sostenerse bajo transformaciones | 2 | fast-check, Hypothesis, custom |
| 3 | **Differential testing** | Compara la salida de dos implementaciones (ej. versión nueva vs. vieja) | 2 | custom, fuzzing frameworks |
| 4 | **Mutation testing** | Muta el código y verifica que los tests detecten la mutación | 3 | Stryker, mutmut, PIT |
| 5 | **Contract testing** | Define contratos entre consumidores y proveedores; ambos lados se validan contra el contrato | 1 | Pact, Schemathesis |

> **Regla:** si el mismo agente escribe el código y el test, al menos uno
> de estos 5 antídotos debe estar presente. Sin excepciones.

---

## Las 3 Advertencias de Uncle Bob

### 1. El problema del oráculo (The Oracle Problem)

> "¿Cómo sabes que el resultado del test es correcto?"

El oráculo es el mecanismo que determina si un test pasa o falla. Si el
oráculo es trivial (un assert hardcodeado), el test no prueba nada —
sólo verifica que el código produce la salida que el código produce.

**Antídoto:** property-based testing (el oráculo es la propiedad, no un
valor hardcodeado), metamorphic testing (el oráculo es la relación entre
transformaciones), differential testing (el oráculo es otra
implementación).

### 2. La objeción de Booch (Metrics don't catch vulnerabilities)

> "Las métricas no capturan vulnerabilidades ni código muerto que un
> ingeniero reconoce de vista." — Grady Booch

La cobertura, los thresholds, los linting rules — ninguno detecta una
vulnerabilidad de diseño. Un 100% de cobertura con tests triviales es
una falsa sensación de seguridad.

**Antídoto:** adversarial review (Nivel 8), mutation testing (Nivel 3),
threat modeling (Nivel 6), SAST/DAST (Nivel 6). La cobertura es
necesaria pero insuficiente.

### 3. Estratifica o el harness se ahoga (Stratify or the harness drowns)

> "Si ejecutas todo en cada commit, el harness se vuelve tan lento que
> nadie lo usa. Si no estratificas, la plétora se convierte en peso
> muerto."

La plétora debe estratificarse en tiers de ejecución con presupuestos de
tiempo estrictos. Si un tier excede su presupuesto, se optimiza o se
mueve al siguiente tier. Sin estratificación, los desarrolladores
ignoran el harness.

**Antídoto:** la estrategia de 4 tiers (pre-commit <10s, PR <10min,
nightly <1h, semanal <4h). Cada nivel tiene su tier. Cada tier tiene su
presupuesto.

---

## BMad Skills — Referencia Completa

Las siguientes skills están disponibles en el proyecto **ecc-enginelegal**
y son aplicables a múltiples niveles del toolkit:

| Skill | Niveles | Descripción |
|-------|---------|-------------|
| `bmad-testarch-trace` | 1, 6, 8 | Genera matriz de trazabilidad y decisión de quality gate. Mapea requisitos → tests. |
| `bmad-qa-generate-e2e-tests` | 1, 5 | Genera tests E2E automatizados para features existentes. |
| `bmad-eval-runner` | 2, 3, 7, 8 | Ejecuta evals de una skill en entorno limpio. Garantiza reproducibilidad. |
| `bmad-review-adversarial-general` | 0, 1, 2, 3, 5, 6, 8 | Review cínico/adversarial. Actúa como red teamer del código y los tests. |
| `bmad-editorial-review-structure` | 0, 4, 8 | Review estructural. Verifica consistencia y completitud de configuración y documentación. |

> **Cómo usarlas:** invoca la skill correspondiente al nivel que estás
> implementando. Por ejemplo, al configurar el Nivel 3 (mutation
> testing), usa `bmad-eval-runner` para validar que los evals corren en
> limpio, y `bmad-review-adversarial-general` para cuestionar si los
> tests sobreviven a mutaciones reales.

---

## Plantillas Incluidas

| Archivo | Propósito |
|---------|-----------|
| [`templates/INVENTARIO.template.md`](./templates/INVENTARIO.template.md) | Plantilla para documentar qué tests ya existen: archivos, suites, casos, comandos, resultados |
| [`templates/MATRIZ_TRAZABILIDAD.template.md`](./templates/MATRIZ_TRAZABILIDAD.template.md) | Plantilla para priorizar gaps pendientes: fases, esfuerzo estimado, orden de ejecución |

> **Uso:** copia estas plantillas a tu proyecto, reemplaza el contenido
> de ejemplo con tu inventario real, y úsalas como living docs que
> actualizas conforme implementas cada nivel.

---

## Principios Rectores

1. **No leas el código. Rodealo de restricciones.** Cada test es una
   restricción que describe comportamiento esperado, no implementación.
2. **La plétora es exhaustiva pero jerárquica.** Nivel 0 → 1 → 2 → 3 →
   4 → 5 → 6 → 7 → 8. Cada nivel asume que el anterior pasa.
3. **Cada nivel tiene un presupuesto de tiempo.** Pre-commit `<10s`, PR
   `<10min`, nightly `<1h`, semanal `<4h`. Si excede, se optimiza o se
   mueve al siguiente tier.
4. **La cobertura es un termómetro, no un fin.** Lo que importa es que
   las restricciones capturen intención.
5. **Cada gap es una restricción pendiente.** Los items faltantes no son
   olvidos — son restricciones identificadas y priorizadas.
6. **Si el mismo agente escribe el código y el test, aplica un
   antídoto.** Property-based, metamorphic, differential, mutation, o
   contract. Sin excepciones.

---

*Toolkit reutilizable. No atado a ningún proyecto, stack ni dominio.
Copia lo que necesites, adapta a tu contexto, y construye tu plétora.*
