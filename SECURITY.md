# Security

This repository is documentation, templates, and methodology — it contains no runnable
application code and no credentials. The security considerations are about **what you put
in it**:

## Rules for contributors

- **No real customer data.** Case studies and examples must use sanitized or synthetic
  data. Before committing, verify no real names, domains, IDs, tickets, or URLs from real
  engagements remain:

  ```bash
  grep -rni "cliente\|@.*\.com\|https\?://" case-study/ templates/ | grep -v example
  ```

- **No proprietary configs.** Test configurations shown here must be generic. Do not paste
  internal CI pipelines, internal URLs, or tool configurations that identify an employer.

- **Secrets scanning.** Run a quick check before pushing:

  ```bash
  grep -rn "api_key\|token\|password\|BEGIN.*PRIVATE KEY" .
  ```

## Reporting

If you spot leaked data in this repo, open a private GitHub Security Advisory rather than
a public issue.
