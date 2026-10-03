# Nivel 5 — No Funcional

> Tests de rendimiento, resiliencia, observabilidad, accesibilidad, visual regression, DR drills.

---

## Tabla de Contenidos

| Test | Tool principal |
|------|----------------|
| [5.1 Benchmarks](#51-benchmarks-con-puerta-de-regresión) | vitest bench / benchmark.js |
| [5.2 Load / stress / soak / spike](#52-load--stress--soak--spike) | Artillery / k6 / Locust |
| [5.3 Profiling y flamegraphs](#53-profiling-y-flamegraphs) | clinic.js / 0x |
| [5.4 Sanitizers](#54-sanitizers) | Node --inspect |
| [5.5 Chaos / fault injection](#55-chaos--fault-injection) | toxiproxy / custom |
| [5.6 Presupuestos de recursos](#56-presupuestos-de-recursos) | size-limit / custom |
| [5.7 Resiliencia](#57-resiliencia) | Custom |
| [5.8 Observability tests](#58-observability-tests) | Jest spy |
| [5.9 DR drills](#59-dr-drills) | Custom |
| [5.10 Accesibilidad](#510-accesibilidad) | axe-core |
| [5.11 i18n / l10n](#511-i18n--l10n) | Custom |
| [5.12 Visual regression](#512-visual-regression) | Playwright / Percy / Chromatic |

---

## 5.1 Benchmarks con puerta de regresión

### Qué es
Benchmarks de funciones críticas con un umbral: falla el CI si hay regresión >X%.

### Por qué importa
Detecta degradación de performance antes de que llegue a producción. Un agente puede introducir una regresión de 10x en una función crítica sin que nadie lo note sin benchmarks.

### Stack
- **JavaScript/TypeScript:** vitest bench o benchmark.js
- **Python:** pytest-benchmark
- **Go:** benchmark builtin (`go test -bench`)
- **Rust:** criterion

### Cómo implementarlo
```bash
# JavaScript
npm install -D vitest
npx vitest bench

# Python
pip install pytest-benchmark
pytest --benchmark-only
```

### Ejemplo de código
```typescript
// bench/userService.bench.ts
import { bench, describe } from 'vitest';
import { calculateTotal } from '../src/services/orderService';

const orders1000 = Array.from({ length: 1000 }, (_, i) => ({
  id: i,
  amount: Math.random() * 1000,
  quantity: Math.floor(Math.random() * 10),
}));

describe('OrderService performance', () => {
  bench('calculateTotal 1000 orders', () => {
    calculateTotal(orders1000);
  });

  bench('calculateTotal 100 orders', () => {
    calculateTotal(orders1000.slice(0, 100));
  });
});
```

### Puerta de regresión
```bash
# Script que compara contra baseline
node scripts/bench-gate.js --threshold 10
# Falla si cualquier benchmark es >10% más lento que el baseline
```

### Cuándo usarlo
- Funciones críticas de cálculo (totales, balances, agregaciones)
- Funciones de parsing/serialización
- Queries de base de datos frecuentes

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| vitest | https://vitest.dev/ | Framework de testing con modo bench |
| benchmark.js | https://benchmarkjs.com/ | Library de benchmarking para JS |
| pytest-benchmark | https://pytest-benchmark.readthedocs.io/ | Benchmarks para Python |
| criterion | https://bheisler.github.io/criterion.rs/ | Benchmarks para Rust |

---

## 5.2 Load / stress / soak / spike

### Qué es
Tests de carga: verificar que el sistema soporta el tráfico esperado sin degradación.

### Tipos
- **Load:** tráfico esperado sostenido
- **Stress:** más allá del límite esperado
- **Soak:** tráfico sostenido por horas (detecta memory leaks)
- **Spike:** picos repentinos de tráfico

### Stack
- **JavaScript:** Artillery o k6
- **Python:** Locust
- **Java:** Gatling
- **Go:** vegeta

### Ejemplo de config (Artillery)
```yaml
# load-test.yml
target: https://api.example.com
phases:
  - name: warmup
    duration: 30
    arrivalRate: 5
  - name: ramp-up
    duration: 60
    arrivalRate: 5
    rampTo: 50
  - name: sustained
    duration: 120
    arrivalRate: 50
  - name: spike
    duration: 30
    arrivalRate: 100
scenarios:
  - weight: 50
    flow: [get: { url: "/api/health" }]
  - weight: 30
    flow:
      - post:
          url: "/api/auth/login"
          json: { email: "test@example.com", password: "test123" }
  - weight: 20
    flow: [get: { url: "/api/users" }]
```

### Ejemplo de config (k6)
```javascript
// load-test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 5 },
    { duration: '1m', target: 50 },
    { duration: '2m', target: 50 },
    { duration: '30s', target: 100 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<2000'],
    http_req_failed: ['rate<0.05'],
  },
};

export default function () {
  http.get('https://api.example.com/api/health');
}
```

### Cuándo usarlo
- Antes de cada release a producción
- Después de cambios en queries de DB o lógica crítica
- Soak test semanal para detectar memory leaks

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| Artillery | https://www.artillery.io/ | Load testing para JS/Node |
| k6 | https://k6.io/ | Load testing moderno (JS) |
| Locust | https://locust.io/ | Load testing en Python |
| Gatling | https://gatling.io/ | Load testing en Scala/Java |
| vegeta | https://github.com/tsenart/vegeta | Load testing en Go |

---

## 5.3 Profiling y flamegraphs

### Qué es
Identificar cuellos de botella de CPU/memoria con visualización de flamegraphs.

### Stack
- **Node.js:** clinic.js, 0x
- **Python:** cProfile, py-spy
- **Go:** pprof
- **Rust:** flamegraph

### Ejecución
```bash
# Node.js
npm install -g clinic
clinic doctor -- node dist/server.js
# Genera reporte HTML con flamegraph

# Python
python -m cProfile -o profile.out myscript.py
pip install snakeviz && snakeviz profile.out

# Go
go test -cpuprofile cpu.prof -bench .
go tool pprof cpu.prof
```

### Cuándo usarlo
- Cuando un endpoint es más lento de lo esperado
- Cuando el consumo de CPU/memoria es alto
- Para optimizar funciones críticas

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| clinic.js | https://clinicjs.org/ | Profiling para Node.js |
| 0x | https://github.com/davidmarkclements/0x | Flamegraphs para Node.js |
| py-spy | https://github.com/benfred/py-spy | Profiling para Python sin modificar código |
| pprof | https://github.com/google/pprof | Profiling para Go |

---

## 5.4 Sanitizers

### Qué es
Detección de memory leaks, use-after-free, undefined behavior.

### Stack
- **Node.js:** `--trace-warnings`, `--trace-deprecation`, `--inspect`
- **C/C++:** ASan, UBSan, TSan, MSan, valgrind
- **Rust:** Miri
- **Go:** `go test -race`

### Ejecución (Node.js)
```bash
node --trace-warnings --trace-deprecation dist/server.js
```

### Cuándo usarlo
- Procesos long-running (cron jobs, workers, daemons)
- Código que maneja buffers/binarios
- Concurrencia (race conditions con `--race`)

---

## 5.5 Chaos / fault injection

### Qué es
Inyectar fallos (red, DB, API externa) y verificar degradación graceful.

### Casos típicos
- Base de datos cae → API debe responder 503, no 500
- Servicio externo (API de pago, email) cae → fallback graceful
- Timeout de servicio → retry con backoff
- Partición de red → modo degradado

### Ejemplo
```typescript
describe('Chaos: external API failure', () => {
  it('order service responde fallback cuando payment API falla', async () => {
    jest.spyOn(paymentService, 'charge').mockRejectedValue(new Error('API 500'));

    const response = await orderService.createOrder({ items: [...], total: 100 });
    expect(response.status).toBe('PENDING');
    expect(response.fallback).toBe(true);
    expect(response.message).toContain('payment will be retried');
  });
});
```

### Cuándo usarlo
- Servicios que dependen de APIs externas
- Sistemas con retry/circuit breaker
- Cron jobs que pueden fallar parcialmente

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| toxiproxy | https://toxiproxy.org/ | Simula fallos de red |
| Chaos Mesh | https://chaos-mesh.org/ | Chaos engineering para Kubernetes |
| Gremlin | https://www.gremlin.com/ | Plataforma comercial de chaos |

---

## 5.6 Presupuestos de recursos

### Qué es
Límites en: bundle size, cold start, memoria, N+1 queries, tamaño de imagen.

### Frontend bundle
```bash
npm install -D size-limit @size-limit/preset-app
```
```json
// .size-limit.json
[
  { "path": "dist/main.js", "limit": "500 KB" },
  { "path": "dist/vendor.js", "limit": "200 KB" }
]
```

### Detección N+1 en ORM
```typescript
describe('N+1 detection', () => {
  it('listar usuarios no hace N+1 queries', async () => {
    const queries: string[] = [];
    jest.spyOn(db, 'query').mockImplementation((sql: string) => {
      queries.push(sql);
      return [];
    });

    await userService.list({ page: 1, limit: 50 });

    // No debería haber 50+ queries (una por usuario)
    expect(queries.length).toBeLessThan(10);
  });
});
```

### Cuándo usarlo
- Frontend: limitar bundle size
- Backend: detectar N+1 queries
- Serverless: limitar cold start time

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| size-limit | https://github.com/ai/size-limit | Presupuestos de bundle para JS |
| bundlephobia | https://bundlephobia.com/ | Mide costo de paquetes npm |

---

## 5.7 Resiliencia

### Qué es
Tests de: timeout, retry, circuit breaker, idempotencia, modo degradado, backpressure.

### Ejemplo
```typescript
describe('Resiliencia del servicio de pagos', () => {
  it('timeout después de 30s', async () => {
    const start = Date.now();
    await expect(paymentService.processLargeBatch(hugeBatch))
      .rejects.toThrow('timeout');
    expect(Date.now() - start).toBeLessThan(35000);
  });

  it('retry 3 veces antes de fallar', async () => {
    let attempts = 0;
    jest.spyOn(httpClient, 'post').mockImplementation(() => {
      attempts++;
      if (attempts < 3) throw new Error('transient');
      return { success: true };
    });

    const result = await service.callWithRetry();
    expect(attempts).toBe(3);
    expect(result.success).toBe(true);
  });

  it('circuit breaker abre después de 5 fallos consecutivos', async () => {
    jest.spyOn(httpClient, 'post').mockRejectedValue(new Error('500'));

    for (let i = 0; i < 5; i++) {
      await service.call().catch(() => {});
    }

    const start = Date.now();
    await service.call().catch(() => {});
    expect(Date.now() - start).toBeLessThan(10); // Fast fail (circuit open)
  });
});
```

### Cuándo usarlo
- Servicios que llaman APIs externas
- Operaciones que pueden fallar transitoriamente
- Sistemas con circuit breakers

---

## 5.8 Observability tests

### Qué es
Verifica que el sistema emite los logs, métricas y traces esperados.

### Ejemplo
```typescript
describe('Audit log en cambio de estado', () => {
  it('cambiar estado de order emite log de auditoría', async () => {
    const logSpy = jest.spyOn(logger, 'info');

    await orderService.changeStatus(orderId, 'COMPLETED', userId);

    expect(logSpy).toHaveBeenCalledWith(
      expect.objectContaining({
        action: 'status_changed',
        entityId: orderId,
        newStatus: 'COMPLETED',
        userId,
      })
    );
  });
});
```

### Cuándo usarlo
- Cambios de estado críticos (orders, payments, permissions)
- Operaciones de auditoría
- Emisión de métricas (Prometheus, StatsD)

---

## 5.9 DR drills

### Qué es
Ensayo de restauración de backup, failover, rollback.

### Script
```bash
#!/bin/bash
# dr-drill.sh
echo "=== DR Drill: DB restore ==="
mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > /tmp/backup.sql
mysql -u$DB_USER -p$DB_PASS -e "CREATE DATABASE dr_test"
mysql -u$DB_USER -p$DB_PASS dr_test < /tmp/backup.sql

# Verificar integridad
TABLES=$(mysql -u$DB_USER -p$DB_PASS dr_test -e "SHOW TABLES" | wc -l)
echo "Tables restored: $TABLES"

# Cleanup
mysql -u$DB_USER -p$DB_PASS -e "DROP DATABASE dr_test"
echo "DR Drill complete"
```

### Cuándo usarlo
- Mensual o semanal
- Antes de migraciones grandes
- Verificar que los backups funcionan

---

## 5.10 Accesibilidad

### Qué es
Verifica WCAG 2.1 AA compliance en la UI.

### Stack
- **Playwright:** @axe-core/playwright
- **Cypress:** cypress-axe
- **Jest:** jest-axe

### Ejemplo (Playwright)
```typescript
import AxeBuilder from '@axe-core/playwright';
import { test, expect } from '@playwright/test';

test('dashboard accesible WCAG AA', async ({ page }) => {
  await page.goto('/dashboard');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});

test('login form accesible', async ({ page }) => {
  await page.goto('/login');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

### Cuándo usarlo
- Páginas principales (login, dashboard, formularios críticos)
- Componentes reutilizables (modales, tabs, dropdowns)
- Antes de cada release

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| axe-core | https://github.com/dequelabs/axe-core | Engine de accesibilidad |
| @axe-core/playwright | https://github.com/dequelabs/axe-core-npm | Integración con Playwright |
| cypress-axe | https://github.com/component-driven/cypress-axe | Integración con Cypress |
| pa11y | https://pa11y.org/ | CLI de accesibilidad |
| Lighthouse | https://github.com/GoogleChrome/lighthouse | Audita performance + a11y |

---

## 5.11 i18n / l10n

### Qué es
Pseudo-localización, layout RTL, formatos de fecha/moneda, tests de traducción.

### Ejemplo
```typescript
describe('i18n', () => {
  it('formato de fecha respeta locale', () => {
    const date = new Date('2026-01-15');
    expect(formatDate(date, 'es-MX')).toBe('15/01/2026');
    expect(formatDate(date, 'en-US')).toBe('01/15/2026');
    expect(formatDate(date, 'ja-JP')).toBe('2026/01/15');
  });

  it('formato de moneda respeta locale', () => {
    expect(formatCurrency(1234.56, 'es-MX')).toMatch(/1,234\.56/);
    expect(formatCurrency(1234.56, 'en-US')).toMatch(/\$1,234\.56/);
  });
});
```

### Cuándo usarlo
- Apps multi-idioma
- Apps con usuarios en múltiples regiones
- Validación de formatos (fecha, moneda, números)

---

## 5.12 Visual regression

### Qué es
Compara screenshots contra baseline. Detecta cambios visuales no intencionales.

### Stack
- **Playwright:** `toHaveScreenshot()`
- **Cypress:** cypress-image-snapshot
- **Comercial:** Percy, Chromatic
- **Self-hosted:** Lost Pixel

### Ejemplo (Playwright)
```typescript
test('navbar visual regression', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page.locator('nav')).toHaveScreenshot('navbar.png', {
    maxDiffPixelRatio: 0.01, // 1% tolerance
  });
});

test('full page regression', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard-full.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.02,
  });
});
```

### Cuándo usarlo
- Componentes UI estables (navbar, sidebar, footer)
- Páginas principales
- Después de cambios de CSS/styling

### Skills y GitHub Projects
| Recurso | URL | Descripción |
|---------|-----|-------------|
| Playwright screenshots | https://playwright.dev/docs/test-snapshots | Built-in visual regression |
| Percy | https://percy.io/ | Visual regression comercial |
| Chromatic | https://www.chromatic.com/ | Visual regression para Storybook |
| Lost Pixel | https://github.com/lost-pixel/lost-pixel | Alternativa self-hosted |
| reg-suit | https://github.com/reg-viz/reg-suit | Comparación de imágenes |
