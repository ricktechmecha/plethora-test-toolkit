# Contributing

This toolkit documents a multi-level testing methodology (unit → mutation → load → security
→ governance) meant to be **reusable across any project or stack**.

## How to contribute

1. Fork and branch (`git checkout -b feat/my-change`).
2. Each level lives in its own numbered `.md` (`00-nivel-0-*.md` …). Keep them
   self-contained — a reader should be able to copy one file and implement that level
   without reading the rest.
3. Templates go in `templates/` and must remain generic (use `INVENTARIO.template.md`
   as reference).
4. Open a PR describing the change.

## Rules

- **Language:** Spanish for methodology docs is fine (audience is LATAM-friendly); file
  names and code snippets stay in English.
- **No vendor lock-in.** When recommending tools, list alternatives per level
  (open-source first).
- **Sanitize case studies** — no real client names, URLs, or data (see SECURITY.md).
- **Cite the source.** When a technique comes from Uncle Bob, mutation-testing literature,
  or a specific framework doc, link it.

## Useful first contributions

- A missing level (e.g., contract testing, chaos engineering).
- Worked examples mapping the methodology to a specific stack (pytest, Jest, Go test).
- Translations of the level docs to English.
