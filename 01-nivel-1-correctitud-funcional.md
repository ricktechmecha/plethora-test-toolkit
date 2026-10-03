# Nivel 1 — Correctitud Funcional (El Núcleo)

> "El código tuvo que pasar el gauntlet completo de restricciones." — Uncle Bob

Este es el corazón del harness. Sin estos tests, no hay nada. Un agente de IA que escribe
código sin tests funcionales es un caos productivo.

---

## Tabla de Contenidos

| Test | Tool |
|------|------|
| [1.1 Unit tests](#11-unit-tests) | Jest / pytest |
| [1.2 Integration tests](#12-integration-tests) | Jest / pytest + testcontainers |
| [1.3 Contract tests](#13-contract-tests-consumer-driven) | Pact |
| [1.4 Acceptance / BDD Gherkin](#14-acceptance--bdd-en-gherkin) | jest-cucumber / pytest-bdd |
| [1.5 End-to-end / system tests](#15-end-to-end--system-tests) | Playwright / Cypress |
| [1.6 Smoke tests](#16-smoke-tests) | Playwright / curl |
| [1.7 Regression suite](#17-regression-suite) | Jest / pytest |
| [1.8 Table-driven / parameterized](#18-table-driven--parameterized-tests) | it.each / pytest.mark.parametrize |
| [1.9 Approval / golden master](#19-approval--golden-master--characterization-testing) | jest-snapshot / approval-tests |
| [1.10 Snapshot tests](#110-snapshot-tests) | Jest / Playwright |

---

## 1.1 Unit tests

### Qué es

Tests aislados, herméticos, sin I/O. Prueban una función o unidad de código en completo
aislamiento, con todas las dependencias mockeadas.

### Por qué importa

Son la base de la pirámide de tests. Rápidos (<1s cada uno), deterministas, y detectan
bugs en la lógica de negocio antes que cualquier otra cosa.

### Stack

- **Node.js / TypeScript**: Jest + ts-jest o Vitest
- **Python**: pytest + pytest-mock
- **Rust**: `cargo test` + `mockall`
- **Go**: `testing` + `testify`

### Cómo implementarlo

```bash
# Node.js / TypeScript
npm install -D jest ts-jest @types/jest

# Python
pip install pytest pytest-mock pytest-cov
```

### Ejemplo de código

```typescript
// src/services/user-service.ts
export function calculateAccountBalance(transactions: Transaction[]): number {
  return transactions.reduce((acc, t) => {
    if (t.type === 'DEPOSIT') return acc + t.amount;
    if (t.type === 'WITHDRAWAL') return acc - t.amount;
    if (t.type === 'REFUND') return acc + t.amount;
    return acc;
  }, 0);
}

// tests/unit/user-service.test.ts
describe('Account balance calculation', () => {
  it('DEPOSIT increases balance', () => {
    const transactions = [{ type: 'DEPOSIT', amount: 100 }];
    expect(calculateAccountBalance(transactions)).toBe(100);
  });

  it('WITHDRAWAL decreases balance', () => {
    const transactions = [
      { type: 'DEPOSIT', amount: 100 },
      { type: 'WITHDRAWAL', amount: 30 },
    ];
    expect(calculateAccountBalance(transactions)).toBe(70);
  });

  it('REFUND increases balance', () => {
    const transactions = [
      { type: 'DEPOSIT', amount: 100 },
      { type: 'WITHDRAWAL', amount: 50 },
      { type: 'REFUND', amount: 50 },
    ];
    expect(calculateAccountBalance(transactions)).toBe(100);
  });
});
```

**Python equivalente**:

```python
# tests/test_user_service.py
import pytest
from src.services.user_service import calculate_account_balance

@pytest.mark.parametrize("transactions, expected", [
    ([{"type": "DEPOSIT", "amount": 100}], 100),
    ([{"type": "DEPOSIT", "amount": 100}, {"type": "WITHDRAWAL", "amount": 30}], 70),
    ([{"type": "DEPOSIT", "amount": 100}, {"type": "WITHDRAWAL", "amount": 50}, {"type": "REFUND", "amount": 50}], 100),
])
def test_calculate_account_balance(transactions, expected):
    assert calculate_account_balance(transactions) == expected
```

### Cuándo usarlo

Siempre. Los unit tests son la base. Apuntar a > 80% coverage de líneas en lógica de
negocio (services, utils, domain logic). No es necesario testear capas de I/O con unit
tests — para eso están los integration tests.

---

## 1.2 Integration tests

### Qué es

Tests con dependencias reales o testcontainers. Verifican que los componentes funcionan
juntos (ej: servicio + base de datos real, servicio + cache real).

### Por qué importa

Los unit tests con mocks no detectan problemas de integración real (ej: query que no
funciona con el schema actual de la base de datos, serialización incorrecta, problemas
de transacción).

### Stack

- **Node.js / TypeScript**: Jest + [testcontainers](https://github.com/testcontainers/testcontainers-node)
- **Python**: pytest + [testcontainers-python](https://github.com/testcontainers/testcontainers-python)

### Cómo implementarlo

```bash
# Node.js
npm install -D @testcontainers/mysql @testcontainers/redis

# Python
pip install testcontainers
```

### Ejemplo de código

```typescript
// tests/integration/user-service.integration.test.ts
import { MySqlContainer } from '@testcontainers/mysql';
import { UserService } from '../../src/services/user-service';

describe('UserService integration with MySQL', () => {
  let container: StartedMySqlContainer;
  let userService: UserService;

  beforeAll(async () => {
    container = await new MySqlContainer().start();
    userService = new UserService({
      host: container.getHost(),
      port: container.getPort(),
      database: container.getDatabase(),
    });
    await userService.runMigrations();
  });

  afterAll(async () => {
    await container.stop();
  });

  it('creates and retrieves a user', async () => {
    const user = await userService.create({ email: 'test@example.com', name: 'Test' });
    const retrieved = await userService.findById(user.id);
    expect(retrieved.email).toBe('test@example.com');
  });

  it('prevents duplicate emails', async () => {
    await userService.create({ email: 'dup@example.com', name: 'First' });
    await expect(userService.create({ email: 'dup@example.com', name: 'Second' }))
      .rejects.toThrow('duplicate');
  });
});
```

### Cuándo usarlo

Cuando un servicio interactúa con una base de datos, cache, message queue, o API externa.
Los integration tests son más lentos que los unit tests pero detectan una clase de bugs
que los mocks nunca encontrarán.

---

## 1.3 Contract tests (consumer-driven)

### Qué es

Tests que verifican que el consumidor (frontend) y el proveedor (backend) acuerdan el
formato del contrato API. El consumidor escribe expectativas, el proveedor verifica que
las cumple.

### Por qué importa

Crítico cuando un agente toca varios servicios. Si el backend cambia el formato de
respuesta, el frontend se rompe silenciosamente.

### Stack

- **Cualquier lenguaje**: [Pact](https://github.com/pact-foundation/pact) (JS, Python, Go, Java, Rust)

### Cómo implementarlo

```bash
# Consumer (frontend)
npm install -D @pact-foundation/pact

# Provider (backend)
npm install -D @pact-foundation/pact
```

### Ejemplo de código

```typescript
// Consumer (frontend) — define el contrato
describe('User API contract', () => {
  it('GET /api/users/:id returns user', async () => {
    return provider.addInteraction({
      uponReceiving: 'a request for a user',
      withRequest: { method: 'GET', path: '/api/users/123' },
      willRespondWith: {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: { id: 123, email: like('test@example.com'), name: like('Test') },
      },
    });
  });
});

// Provider (backend) — verifica que cumple el contrato
describe('Pact verification', () => {
  it('validates user endpoint', async () => {
    const res = await request.get('/api/users/123');
    expect(res.status).toBe(200);
    expect(res.body.id).toBe(123);
    expect(res.body.email).toBeDefined();
  });
});
```

### Cuándo usarlo

Cuando hay un frontend y un backend separados, o cuando hay microservicios que se
comunican vía API. Si todo está en un monolito, los integration tests pueden ser
suficientes.

---

## 1.4 Acceptance / BDD en Gherkin

### Qué es

Tests de aceptación escritos en lenguaje natural (Gherkin: Given/When/Then) que describen
comportamiento esperado. **Esto es lo que Uncle Bob sí revisa a mano.**

### Por qué importa

Es la conexión entre requisitos y tests. Un stakeholder no técnico puede leer los Gherkin
y validar que el comportamiento es correcto. Si el agente escribe un Gherkin sutilmente
equivocado, una revisión humana lo puede detectar.

### Stack

- **Node.js / TypeScript**: [jest-cucumber](https://github.com/namastegit/jest-cucumber) o [@cucumber/cucumber](https://github.com/cucumber/cucumber-js)
- **Python**: [pytest-bdd](https://github.com/pytest-dev/pytest-bdd) o [behave](https://github.com/behave/behave)

### Cómo implementarlo

```bash
# Node.js
npm install -D jest-cucumber

# Python
pip install pytest-bdd
```

### Ejemplo de código

```gherkin
# tests/features/order-checkout.feature
Feature: Order checkout
  As a customer
  I want to checkout my shopping cart
  So that I can complete my purchase

  Scenario: Customer checks out with valid payment
    Given a shopping cart with 3 items totaling $150
    And a valid payment method
    When the customer submits the checkout
    Then the order is created with status "PENDING_PAYMENT"
    And the payment is processed
    And the order status changes to "CONFIRMED"
    And a confirmation email is sent

  Scenario: Customer checks out with invalid payment
    Given a shopping cart with 2 items totaling $80
    And an invalid payment method
    When the customer submits the checkout
    Then the order is created with status "PENDING_PAYMENT"
    And the payment fails
    And the order status changes to "PAYMENT_FAILED"
    And no confirmation email is sent
```

```typescript
// tests/features/order-checkout.steps.ts
import { defineFeature, loadFeature } from 'jest-cucumber';
import { OrderService } from '../../src/services/order-service';

const feature = loadFeature('tests/features/order-checkout.feature');

defineFeature(feature, (test) => {
  let orderService: OrderService;
  let result: any;

  beforeEach(() => {
    orderService = new OrderService();
  });

  test('Customer checks out with valid payment', ({ given, when, then }) => {
    given('a shopping cart with 3 items totaling $150', () => {
      // setup cart
    });

    given('a valid payment method', () => {
      // setup payment
    });

    when('the customer submits the checkout', async () => {
      result = await orderService.checkout(cartId, paymentMethod);
    });

    then('the order is created with status "PENDING_PAYMENT"', () => {
      expect(result.status).toBe('PENDING_PAYMENT');
    });

    then('the order status changes to "CONFIRMED"', () => {
      expect(result.finalStatus).toBe('CONFIRMED');
    });
  });
});
```

### Cuándo usarlo

Para features que tienen valor de negocio claro y que un stakeholder puede validar. No
todos los tests necesitan ser Gherkin — los tests técnicos (utils, helpers) pueden ser
unit tests normales. Usa Gherkin para acceptance criteria de stories.

---

## 1.5 End-to-end / system tests

### Qué es

Tests que ejercitan el sistema completo, de punta a punta, incluyendo UI, API, base de
datos, y servicios externos (o mocks de ellos).

### Por qué importa

Los E2E tests son los que más se acercan a lo que el usuario real hace. Detectan problemas
que los unit e integration tests no pueden: integración frontend-backend, routing,
rendering, flujos completos.

### Stack

- **Web**: [Playwright](https://github.com/microsoft/playwright) o [Cypress](https://github.com/cypress-io/cypress)
- **API**: [supertest](https://github.com/ladjs/supertest) (Node.js) o [httpx](https://github.com/encode/httpx) (Python)

### Cómo implementarlo

```bash
# Playwright
npm install -D @playwright/test
npx playwright install

# Cypress
npm install -D cypress
```

### Ejemplo de código

```typescript
// tests/e2e/order-flow.spec.ts
import { test, expect } from '@playwright/test';

test('customer can place an order', async ({ page }) => {
  await page.goto('http://localhost:3000');

  // Login
  await page.fill('[data-testid=email]', 'customer@example.com');
  await page.fill('[data-testid=password]', 'password123');
  await page.click('[data-testid=login-button]');

  // Add items to cart
  await page.click('[data-testid=product-1]');
  await page.click('[data-testid=add-to-cart]');
  await page.click('[data-testid=product-2]');
  await page.click('[data-testid=add-to-cart]');

  // Checkout
  await page.click('[data-testid=checkout]');
  await page.fill('[data-testid=card-number]', '4242424242424242');
  await page.fill('[data-testid=card-expiry]', '12/28');
  await page.fill('[data-testid=card-cvc]', '123');
  await page.click('[data-testid=submit-order]');

  // Verify
  await expect(page.locator('[data-testid=order-confirmation]')).toBeVisible();
  await expect(page.locator('[data-testid=order-status]')).toHaveText('CONFIRMED');
});
```

### Cuándo usarlo

Para flujos críticos de negocio (login, checkout, registro, flujos principales). Los E2E
tests son lentos y frágiles — no los uses para todo. Apunta a 10-20 E2E tests que cubran
los happy paths y los edge cases más importantes.

---

## 1.6 Smoke tests

### Qué es

Tests mínimos que verifican que el sistema está vivo y respondiendo. Corren después de
cada deploy.

### Por qué importa

Un deploy que rompe el health check debe detectarse en segundos, no en minutos. Los smoke
tests son la primera línea de defensa post-deploy.

### Stack

- **Cualquier lenguaje**: curl + script, o Playwright para smoke tests de UI

### Ejemplo de código

```typescript
// tests/smoke/smoke.spec.ts
import { test, expect } from '@playwright/test';

test('health endpoint responds', async ({ request }) => {
  const res = await request.get('/api/health');
  expect(res.ok()).toBeTruthy();
  const body = await res.json();
  expect(body.status).toBe('ok');
});

test('homepage loads', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toBeVisible();
});

test('login page loads', async ({ page }) => {
  await page.goto('/login');
  await expect(page.locator('[data-testid=login-button]')).toBeVisible();
});
```

### Cuándo usarlo

Después de cada deploy a cualquier entorno. Los smoke tests deben correr en < 30 segundos
y dar confianza de que el sistema está operativo.

---

## 1.7 Regression suite

### Qué es

Tests que se agregan específicamente para prevenir que un bug ya encontrado y arreglado
vuelva a ocurrir.

### Por qué importa

Cada bug que se encuentra es un test que faltaba. El bug se convierte en test de regresión
y nunca vuelve a pasar desapercibido.

### Ejemplo de código

```typescript
// tests/regression/issue-123-null-pointer.test.ts
// Regression test for issue #123: UserService.findById crashes on null ID
describe('Regression: issue #123', () => {
  it('findById handles null ID gracefully', async () => {
    await expect(userService.findById(null as any)).rejects.toThrow('ID is required');
    // Before fix: this threw a NullPointerException
  });
});
```

### Cuándo usarlo

Cada vez que se arregla un bug, se escribe un test de regresión. El test se nombra con
referencia al issue/ticket para trazabilidad.

---

## 1.8 Table-driven / parameterized tests

### Qué es

Un solo test que se ejecuta con múltiples conjuntos de datos (tabla de inputs y outputs
esperados).

### Por qué importa

Reduce duplicación de tests y hace fácil agregar nuevos casos. Un agente de IA puede
generar 50 casos de test en una tabla en lugar de 50 funciones de test separadas.

### Ejemplo de código

```typescript
// Node.js / TypeScript
describe('EmailValidator', () => {
  it.each([
    ['user@example.com', true],
    ['user.name@example.com', true],
    ['user+tag@example.co.uk', true],
    ['', false],
    ['not-an-email', false],
    ['@example.com', false],
    ['user@', false],
    ['user@@example.com', false],
    ['user@example', false],
  ])('validate("%s") returns %s', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });
});
```

```python
# Python
import pytest
from src.validators import validate_email

@pytest.mark.parametrize("email, expected", [
    ("user@example.com", True),
    ("user.name@example.com", True),
    ("user+tag@example.co.uk", True),
    ("", False),
    ("not-an-email", False),
    ("@example.com", False),
    ("user@", False),
    ("user@@example.com", False),
    ("user@example", False),
])
def test_validate_email(email, expected):
    assert validate_email(email) == expected
```

### Cuándo usarlo

Siempre que tengas múltiples casos del mismo test con diferentes inputs. Es el patrón
preferido para tests de validación, parsing, transformación, y cualquier función pura.

---

## 1.9 Approval / golden master / characterization testing

### Qué es

Tests que capturan la salida actual de un sistema como "golden master" y verifican que
futuras ejecuciones producen la misma salida. Útil para código legacy sin tests donde no
sabes qué debería producir pero sabes que no debería cambiar.

### Por qué importa

Para código legacy sin especificación, el golden master es la única forma de refactorizar
con seguridad. Capturas el comportamiento actual (correcto o no) y verificas que no cambia.

### Stack

- **Node.js**: [jest-snapshot](https://jestjs.io/docs/snapshot-testing) o [approval-tests](https://github.com/approvals/ApprovalTests.JavaScript)
- **Python**: [pytest approval](https://github.com/approvals/ApprovalTests.Python) o [syrupy](https://github.com/syrupy-project/syrupy)

### Ejemplo de código

```typescript
// tests/approval/report-generator.test.ts
describe('ReportGenerator (golden master)', () => {
  it('generates consistent monthly report', async () => {
    const report = await reportGenerator.generate({
      month: '2026-01',
      format: 'json',
    });
    // First run: saves snapshot
    // Subsequent runs: compares against saved snapshot
    expect(report).toMatchInlineSnapshot();
  });
});
```

### Cuándo usarlo

Para código legacy sin tests, para outputs complejos (reportes, PDFs, JSON anidado), y
para sistemas donde no hay un oráculo claro pero la salida debe ser determinista.

---

## 1.10 Snapshot tests

### Qué es

Tests que guardan una representación serializada de la salida (JSON, HTML, componente UI)
y verifican que no cambia entre ejecuciones.

### Por qué importa

Para UI components y outputs serializados, los snapshots detectan cambios no intencionales
con mínimo código de test.

### Ejemplo de código

```typescript
// tests/snapshot/user-card.test.tsx
import { render } from '@testing-library/react';
import { UserCard } from '../../src/components/UserCard';

describe('UserCard snapshot', () => {
  it('renders consistently', () => {
    const { container } = render(
      <UserCard user={{ id: 1, name: 'Alice', email: 'alice@example.com' }} />
    );
    expect(container).toMatchSnapshot();
  });
});
```

### Cuándo usarlo

Para componentes UI y outputs serializados donde el formato exacto importa. **Cuidado:**
los snapshots son frágiles — cualquier cambio cosmético rompe el test. Úsalos para
componentes estables, no para componentes en desarrollo activo.

---

## Skills y GitHub Projects

### BMad Skills relevantes

- **`bmad-qa-generate-e2e-tests`** — genera tests E2E automatizados para features
  existentes. Útil cuando tienes código funcionando pero sin cobertura E2E. El skill
  analiza la feature y genera tests de API y E2E automáticamente.
- **`bmad-testarch-trace`** — genera matriz de trazabilidad entre requisitos y tests.
  Útil para verificar que cada Gherkin scenario tiene un test correspondiente y que cada
  test tiene un requisito de origen.

### GitHub Projects y npm packages

| Tool | Repo / Package | Instalación |
|------|----------------|-------------|
| Jest | [jestjs/jest](https://github.com/jestjs/jest) | `npm i -D jest ts-jest` |
| Vitest | [vitest-dev/vitest](https://github.com/vitest-dev/vitest) | `npm i -D vitest` |
| pytest | [pytest-dev/pytest](https://github.com/pytest-dev/pytest) | `pip install pytest` |
| Playwright | [microsoft/playwright](https://github.com/microsoft/playwright) | `npm i -D @playwright/test` |
| Cypress | [cypress-io/cypress](https://github.com/cypress-io/cypress) | `npm i -D cypress` |
| Pact | [pact-foundation/pact-js](https://github.com/pact-foundation/pact-js) | `npm i -D @pact-foundation/pact` |
| jest-cucumber | [namastegit/jest-cucumber](https://github.com/namastegit/jest-cucumber) | `npm i -D jest-cucumber` |
| pytest-bdd | [pytest-dev/pytest-bdd](https://github.com/pytest-dev/pytest-bdd) | `pip install pytest-bdd` |
| testcontainers (Node) | [testcontainers/testcontainers-node](https://github.com/testcontainers/testcontainers-node) | `npm i -D @testcontainers/mysql` |
| testcontainers (Python) | [testcontainers/testcontainers-python](https://github.com/testcontainers/testcontainers-python) | `pip install testcontainers` |
| supertest | [ladjs/supertest](https://github.com/ladjs/supertest) | `npm i -D supertest` |
