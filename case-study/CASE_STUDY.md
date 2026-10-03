# Case Study — Plethora Applied to a Production Multi-Tenant SaaS

> Sanitized summary of a real-world application of this toolkit.
> The product and codebase are private (commercial), but the
> methodology and metrics below are real and verifiable by the author.

## Context

A production multi-tenant SaaS platform (TypeScript backend, Next.js
frontend, ~145 data models, 100+ API route modules) built and operated
by a small team over ~8 months. The Plethora methodology was applied
incrementally from level 0 upwards.

## Results by level

| Level | Evidence |
|-------|----------|
| 0 — Cheap gates | Lint clean under project config, conventional commits, secret scanning (gitleaks) in workflow |
| 1 — Functional correctness | **111 Jest suites / 1,949 backend tests passing**; ~60 Playwright E2E specs; contract tests **8/8 passing** |
| 2 — Automatic generation | 9 property-based test properties, ~5,500 generated cases per run |
| 3 — Testing the tests | Mutation testing with Stryker, score **~74%** on core modules |
| 4 — Structure & architecture | dependency-cruiser with **0 violations** across the backend module graph |
| 5 — Non-functional | Staging smoke suite **133/133 passing** against deployed environment |
| 6 — Security & privacy | Semgrep findings classified and remediated; JWT + TOTP 2FA covered by tests |
| 7 — Data & AI | LLM-facing endpoints under contract tests |
| 8 — Process & governance | Requirement→test traceability matrix maintained per epic/story |

## What the levels actually caught

- **Level 3 (mutation):** killed mutants revealed that several "green"
  unit tests asserted nothing meaningful about boundary conditions —
  the highest-value level relative to effort.
- **Level 4 (architecture):** cycle detection surfaced two circular
  dependencies introduced by generated code that would have become
  import-order bugs.
- **Level 8 (traceability):** the requirement→test matrix made gaps
  visible before release, not after incidents.

## How to reproduce on your project

1. Start at level 0 — gates are cheap and catch the most per hour spent.
2. Write the traceability matrix (level 8) early; it tells you *which*
   of the lower levels matter for each feature.
3. Mutation-test only your highest-risk modules first; full-repo
   mutation scores are expensive and slow.
