# Nivel 6 — Seguridad y Privacidad

> Si tu producto maneja datos de usuarios, seguridad no es opcional. Multi-tenant, authn/authz, inyección, dependencias vulnerables.

---

## Tabla de Contenidos

| Test | Tool principal |
|------|----------------|
| [6.1 Threat model (STRIDE)](#61-threat-model-stride) | Manual |
| [6.2 SCA / dependencias](#62-sca--vulnerabilidades-de-dependencias) | osv-scanner / trivy |
| [6.3 DAST](#63-dast) | OWASP ZAP / Nuclei |
| [6.4 Matriz authn/authz](#64-matriz-authnauthz) | Custom |
| [6.5 Suites de inyección](#65-suites-de-inyección) | Custom |
| [6.6 Crypto misuse](#66-crypto-misuse-detection) | eslint-plugin-security |
| [6.7 Rate limiting](#67-rate-limiting-y-anti-abuso) | Custom |
| [6.8 IaC scanning](#68-iac-scanning) | checkov / tfsec |
| [6.9 SBOM](#69-sbom-licencias-y-procedencia) | syft |
| [6.10 Privacidad](#610-privacidad) | Custom |

---

## 6.1 Threat model (STRIDE)

### Qué es
Modelo de amenazas estructurado: **S**poofing, **T**ampering, **R**epudiation, **I**nformation Disclosure, **D**enial of Service, **E**levation of Privilege.

### Por qué importa
Sin un threat model, no sabes qué estás protegiendo ni contra qué. Debe ser un artefacto versionado y revisado.

### Template
```markdown
# docs/security/threat-model.md

## Sistema: [nombre]

## Superficie de ataque
- Endpoints API
- UI
- Base de datos
- Servicios externos

## STRIDE

| Categoría | Amenaza | Mitigación | Estado |
|-----------|---------|------------|--------|
| Spoofing | JWT token theft | Tokens con expiración corta | ✅ |
| Spoofing | OAuth token replay | Validación con provider | ✅ |
| Tampering | Modificar datos bypassando validación | Server-side validation | ✅ |
| Repudiation | Usuario niega acción | Audit log inmutable | ✅ |
| Info Disclosure | Cross-tenant access | Filtro de organizationId | ✅ |
| DoS | Upload de archivos grandes | File size limit | ✅ |
| Elevation | USER accede a endpoints ADMIN | Role check middleware | ✅ |
```

### Cuándo usarlo
- Al diseñar un sistema nuevo
- Al agregar endpoints nuevos
- Revisión semestral

---

## 6.2 SCA / vulnerabilidades de dependencias

### Qué es
Detecta paquetes con vulnerabilidades conocidas (CVEs).

### Stack
- **Universal:** osv-scanner, trivy, dependabot
- **JavaScript:** npm audit
- **Python:** pip-audit, safety
- **Go:** govulncheck

### Ejecución
```bash
# osv-scanner (universal)
osv-scanner -r package.json
osv-scanner -r requirements.txt
osv-scanner -r go.mod

# trivy
trivy fs .

# npm audit
npm audit --audit-level=high

# Python
pip-audit
safety check

# Go
govulncheck ./...
```

### CI integration
```bash
# Script que corre nightly
#!/bin/bash
osv-scanner -r . --format=json -o sca-report.json
if [ $? -ne 0 ]; then
  echo "ALERT: Vulnerable dependencies found"
  # Enviar notificación
  exit 1
fi
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| osv-scanner | https://github.com/google/osv-scanner | Scanner universal de CVEs |
| trivy | https://github.com/aquasecurity/trivy | Scanner de imágenes + FS |
| Dependabot | https://docs.github.com/en/code-security/dependabot | PRs automáticos para fixes |
| pip-audit | https://github.com/pypa/pip-audit | SCA para Python |
| govulncheck | https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck | SCA para Go |

---

## 6.3 DAST

### Qué es
Dynamic Application Security Testing — escanea la aplicación en ejecución.

### Stack
- **OWASP ZAP** — scanner comprehensivo
- **Nuclei** — templates rápidas
- **Burp Suite** — comercial

### Ejecución
```bash
# OWASP ZAP
zap-cli quick-scan --spider https://api.example.com
zap-cli active-scan https://api.example.com/api/login

# Nuclei
nuclei -u https://api.example.com -t cves/
nuclei -u https://api.example.com -t exposures/
```

### Cuándo usarlo
- Nightly contra staging
- Antes de cada release
- Después de agregar endpoints nuevos

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| OWASP ZAP | https://www.zaproxy.org/ | DAST comprehensivo |
| Nuclei | https://github.com/projectdiscovery/nuclei | DAST con templates |
| Burp Suite | https://portswigger.net/burp | Comercial, muy potente |

---

## 6.4 Matriz authn/authz

### Qué es
Todo rol × todo endpoint, generada exhaustivamente. **Crítico para multi-tenant.**

### Ejemplo de código
```typescript
const ROLES = ['SUPER_ADMIN', 'ADMIN', 'USER', 'VIEWER'];

const ENDPOINTS = [
  { method: 'GET', path: '/api/orders', allowed: ['SUPER_ADMIN', 'ADMIN', 'USER', 'VIEWER'] },
  { method: 'POST', path: '/api/orders', allowed: ['SUPER_ADMIN', 'ADMIN', 'USER'] },
  { method: 'DELETE', path: '/api/orders/:id', allowed: ['SUPER_ADMIN', 'ADMIN'] },
  { method: 'GET', path: '/api/admin/users', allowed: ['SUPER_ADMIN'] },
  { method: 'PUT', path: '/api/admin/settings', allowed: ['SUPER_ADMIN'] },
  // ... todos los endpoints
];

ENDPOINTS.forEach(({ method, path, allowed }) => {
  ROLES.forEach(role => {
    test(`${role} ${method} ${path} → ${allowed.includes(role) ? '200' : '403'}`, async () => {
      const token = await getTokenForRole(role);
      const res = await request[method.toLowerCase()](path)
        .set('Authorization', `Bearer ${token}`);
      if (allowed.includes(role)) {
        expect(res.status).not.toBe(403);
      } else {
        expect(res.status).toBe(403);
      }
    });
  });
});
```

### IDOR + Multi-tenant isolation
```typescript
describe('Multi-tenant isolation', () => {
  it('usuario de org A no puede ver recurso de org B', async () => {
    const tokenOrgA = await getTokenForOrg('org-a');
    const resourceOrgB = await createResourceInOrg('org-b');

    const res = await request.get(`/api/resources/${resourceOrgB.id}`)
      .set('Authorization', `Bearer ${tokenOrgA}`);

    expect(res.status).toBe(404); // No 403 (no revelar existencia)
  });
});
```

### Cuándo usarlo
- **Siempre** para sistemas multi-tenant
- Cuando hay roles diferenciados
- Después de agregar endpoints nuevos

---

## 6.5 Suites de inyección

### Qué es
Suites comprehensivas de: SQLi, XSS, SSRF, path traversal, deserialización insegura, XXE.

### Payloads
```typescript
const SQLI_PAYLOADS = [
  "' OR '1'='1",
  "'; DROP TABLE users; --",
  "' UNION SELECT * FROM users --",
  "1; SELECT * FROM dual",
  "' OR 1=1#",
  "admin'--",
  "' OR '1'='1' --",
  "' OR ''='",
  "1' OR '1'='1",
  "' OR 1=1 --",
];

const XSS_PAYLOADS = [
  '<script>alert(1)</script>',
  '"><script>alert(1)</script>',
  '<img src=x onerror=alert(1)>',
  'javascript:alert(1)',
  '<svg onload=alert(1)>',
  '"><iframe src=javascript:alert(1)>',
  '<body onload=alert(1)>',
];

const PATH_TRAVERSAL = [
  '../../../etc/passwd',
  '..\\..\\..\\windows\\win.ini',
  '....//....//....//etc/passwd',
  '%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd',
];

const SSRF_PAYLOADS = [
  'http://localhost:8080/admin',
  'http://169.254.169.254/latest/meta-data/',  // AWS metadata
  'http://[::1]/',
  'file:///etc/passwd',
];
```

### Test pattern
```typescript
SQLI_PAYLOADS.forEach(payload => {
  test(`SQLi: ${payload} no funciona`, async () => {
    const res = await request.post('/api/login').send({ email: payload, password: 'test' });
    expect(res.status).not.toBe(200);
    expect(res.body).not.toContainProperty('token');
  });
});
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| OWASP Testing Guide | https://owasp.org/www-project-web-security-testing-guide/ | Guía comprehensiva |
| PayloadsAllTheThings | https://github.com/swisskyrepo/PayloadsAllTheThings | Payloads para todo |

---

## 6.6 Crypto misuse detection

### Stack
- **JavaScript:** eslint-plugin-security
- **Python:** bandit
- **Go:** gosec

### Config (ESLint)
```json
{
  "plugins": ["security"],
  "rules": {
    "security/detect-object-injection": "warn",
    "security/detect-non-literal-regexp": "error",
    "security/detect-non-literal-fs-filename": "warn",
    "security/detect-unsafe-regex": "error",
    "security/detect-pseudoRandomBytes": "error"
  }
}
```

### Verificaciones
- JWT secret >= 32 caracteres
- bcrypt rounds >= 10
- No usar `Math.random()` para crypto
- No usar `eval()`
- TLS 1.2+ mínimo

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| eslint-plugin-security | https://github.com/eslint-community/eslint-plugin-security | Reglas de seguridad para JS |
| bandit | https://github.com/PyCQA/bandit | SAST para Python |
| gosec | https://github.com/securego/gosec | SAST para Go |

---

## 6.7 Rate limiting y anti-abuso

### Test
```typescript
describe('Rate limiting', () => {
  it('login rate limited después de N intentos', async () => {
    for (let i = 0; i < 10; i++) {
      await request.post('/api/auth/login').send({ email: 'test@test.com', password: 'wrong' });
    }
    const res = await request.post('/api/auth/login').send({ email: 'test@test.com', password: 'wrong' });
    expect(res.status).toBe(429);
  });
});
```

---

## 6.8 IaC scanning

### Qué es
Escanea Docker Compose, Terraform, Kubernetes por misconfigurations.

### Stack
- **checkov** — universal (Terraform, K8s, Docker, CloudFormation)
- **tfsec** — Terraform específico
- **kube-score** — Kubernetes específico

### Ejecución
```bash
checkov -f docker-compose.yml
checkov -d terraform/
tfsec .
kube-score deploy/*.yaml
```

### Verificaciones
- No exponer puertos innecesarios
- No correr como root
- No hardcodear secrets
- Usar imágenes con tag específico (no `:latest`)

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| checkov | https://github.com/bridgecrewio/checkov | IaC scanning universal |
| tfsec | https://github.com/aquasecurity/tfsec | Terraform scanning |
| kube-score | https://github.com/zegl/kube-score | Kubernetes scoring |

---

## 6.9 SBOM, licencias y procedencia

### Qué es
Software Bill of Materials — inventario completo de dependencias con versiones y licencias.

### Stack
- **syft** — genera SBOM
- **cosign** — firma contenedores
- **grype** — analiza SBOM por vulnerabilidades

### Ejecución
```bash
syft . -o json > sbom.json
syft . -o cyclonedx-json > sbom.cdx.json
grype sbom:./sbom.json
```

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| syft | https://github.com/anchore/syft | Generación de SBOM |
| grype | https://github.com/anchore/grype | Análisis de vulnerabilidades sobre SBOM |
| cosign | https://github.com/sigstore/cosign | Firma de contenedores |

---

## 6.10 Privacidad

### Qué es
Trazado de flujos de PII, tests de retención y borrado, gating de consentimiento.

### Tests
```typescript
describe('Privacy — user deletion', () => {
  it('al borrar usuario, sus datos PII se anonimizan', async () => {
    const user = await createTestUser({ email: 'real@email.com', name: 'Real Name' });
    await userService.delete(user.id);

    const deleted = await db.user.findUnique({ where: { id: user.id } });
    expect(deleted.email).toBeNull(); // o hash
    expect(deleted.name).toBeNull();
  });
});

describe('Privacy — no logear PII', () => {
  it('logs no contienen emails completos', async () => {
    const logSpy = jest.spyOn(logger, 'info');
    await userService.process({ email: 'secret@email.com' });

    const logs = logSpy.mock.calls.map(c => JSON.stringify(c));
    logs.forEach(log => {
      expect(log).not.toContain('secret@email.com');
    });
  });
});
```

### Cuándo usarlo
- GDPR/CCPA compliance
- Sistemas con datos sensibles (salud, finanzas, PII)
- Funcionalidad de "derecho al olvido"
