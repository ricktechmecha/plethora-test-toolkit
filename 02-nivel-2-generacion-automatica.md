# Nivel 2 — Generación Automática de Casos

> "El agente escribe la propiedad, la máquina genera los miles de casos." — Uncle Bob

Este es el **verdadero apalancamiento** con agentes de IA. En lugar de escribir 100 tests
manuales, el agente escribe 1 propiedad y la máquina genera 10,000 casos.

---

## Tabla de Contenidos

| Test | Tool |
|------|------|
| [2.1 Property-based testing](#21-property-based-testing) | fast-check / Hypothesis |
| [2.2 Stateful / model-based](#22-stateful--model-based-property-testing) | fast-check |
| [2.3 Metamorphic testing](#23-metamorphic-testing) | Custom |
| [2.4 Differential testing](#24-differential-testing) | Custom |
| [2.5 Fuzzing guiado por cobertura](#25-fuzzing-guiado-por-cobertura) | libFuzzer / Atheris |
| [2.6 Grammar-based fuzzing](#26-grammar-based-fuzzing) | Custom / fuzzingbook |
| [2.7 API fuzzing desde esquema](#27-api-fuzzing-desde-el-esquema) | Schemathesis / restler |
| [2.8 Design by contract](#28-design-by-contract) | Custom / contracts |
| [2.9 Verificación formal](#29-verificación-formal--tla-alloy) | TLA+ / Alloy |
| [2.10 Deterministic simulation](#210-deterministic-simulation-testing) | madsim / Antithesis |

---

## 2.1 Property-based testing

### Qué es

El agente define una **propiedad** (invariante que debe cumplirse para cualquier input
válido), y la librería genera miles de casos aleatorios intentando romperla.

### Por qué importa

**El problema del oráculo:** un agente que escribe código y su test tiende a producir un
test que *confirma el bug*. Property-based testing es uno de los antídotos estructurales —
la máquina explora casos que el agente no imaginó.

### Stack

- **Node.js / TypeScript**: [fast-check](https://github.com/dubzzz/fast-check)
- **Python**: [Hypothesis](https://github.com/HypothesisWorks/hypothesis)
- **Rust**: [proptest](https://github.com/AltSysrq/proptest) o [quickcheck](https://github.com/BurntSushi/quickcheck)

### Cómo implementarlo

```bash
# Node.js / TypeScript
npm install -D fast-check

# Python
pip install hypothesis
```

### Ejemplo de código

```typescript
import fc from 'fast-check';

describe('OrderService — property-based', () => {
  it('total is always non-negative for valid order items', () => {
    fc.assert(
      fc.property(
        fc.array(
          fc.record({
            productId: fc.string({ minLength: 1 }),
            quantity: fc.nat(1000),
            unitPrice: fc.float({ min: 0, max: 10000, noNaN: true }),
          }),
          { maxLength: 100 }
        ),
        (items) => {
          const total = calculateOrderTotal(items);
          expect(total).toBeGreaterThanOrEqual(0);
        }
      ),
      { numRuns: 1000 }
    );
  });

  it('order total = sum of (quantity * unitPrice) for all items', () => {
    fc.assert(
      fc.property(
        fc.array(
          fc.record({
            productId: fc.string(),
            quantity: fc.nat(1000),
            unitPrice: fc.float({ min: 0, max: 1000, noNaN: true }),
          })
        ),
        (items) => {
          const total = calculateOrderTotal(items);
          const expected = items.reduce((sum, i) => sum + i.quantity * i.unitPrice, 0);
          expect(total).toBeCloseTo(expected, 2);
        }
      )
    );
  });
});
```

**Python equivalente**:

```python
from hypothesis import given, strategies as st

@given(st.lists(
    st.fixed_keys({
        "product_id": st.text(min_size=1),
        "quantity": st.integers(min_value=0, max_value=1000),
        "unit_price": st.floats(min_value=0, max_value=10000, allow_nan=False),
    }),
    max_size=100,
))
def test_order_total_non_negative(items):
    total = calculate_order_total(items)
    assert total >= 0
```

### Cuándo usarlo

Para cualquier función que tenga invariantes matemáticos o lógicos. Especialmente útil
para: parsers, validadores, cálculos financieros, algoritmos de ordenamiento, estructuras
de datos. **Prioridad máxima** — es uno de los antídotos contra el problema del oráculo.

---

## 2.2 Stateful / model-based property testing

### Qué es

Modela el sistema como una máquina de estados. Genera secuencias aleatorias de
transiciones y verifica que los invariantes se cumplen después de cada transición.

### Por qué importa

Detecta bugs que solo aparecen después de secuencias específicas de operaciones (ej:
crear → editar → eliminar → crear de nuevo → el ID colisiona).

### Ejemplo de código

```typescript
import fc from 'fast-check';

// Order state machine:
// PENDING → CONFIRMED → SHIPPED → DELIVERED
// PENDING → CANCELLED
// CONFIRMED → CANCELLED
// Any state → CANCELLED (except DELIVERED)

const commands = [
  fc.constant({ kind: 'confirm' }),
  fc.constant({ kind: 'ship' }),
  fc.constant({ kind: 'deliver' }),
  fc.constant({ kind: 'cancel' }),
];

describe('Order state machine', () => {
  it('respects valid transitions', () => {
    fc.assert(
      fc.property(
        fc.array(fc.oneof(...commands), { maxLength: 20 }),
        (commandSequence) => {
          let state = 'PENDING';
          for (const cmd of commandSequence) {
            const prevState = state;
            state = applyTransition(state, cmd.kind);
            expect(isValidTransition(prevState, state)).toBe(true);
          }
        }
      )
    );
  });

  it('DELIVERED orders cannot be cancelled', () => {
    fc.assert(
      fc.property(
        fc.array(fc.oneof(...commands), { maxLength: 20 }),
        (commandSequence) => {
          let state = 'PENDING';
          for (const cmd of commandSequence) {
            if (state === 'DELIVERED' && cmd.kind === 'cancel') {
              expect(() => applyTransition(state, cmd.kind)).toThrow();
              return;
            }
            state = applyTransition(state, cmd.kind);
          }
        }
      )
    );
  });
});
```

### Cuándo usarlo

Para sistemas con estado (órdenes, sesiones, workflows, máquinas de estados finitos).
Especialmente útil cuando hay transiciones complejas que son fáciles de romper con
secuencias inesperadas.

---

## 2.3 Metamorphic testing

### Qué es

Cuando no existe un oráculo (no sabes la respuesta correcta), verificas **relaciones entre
entradas y salidas**. Ej: `sort(shuffle(x)) == sort(x)`.

### Por qué importa

Para funciones complejas (ML, búsqueda, ranking, transformaciones) donde no siempre sabes
la respuesta exacta, pero sabes relaciones que deben cumplirse.

### Ejemplo de código

```typescript
describe('SearchService — metamorphic testing', () => {
  it('search("laptop computer") = search("computer laptop") (same set, order-independent)', async () => {
    const res1 = await searchService.search('laptop computer');
    const res2 = await searchService.search('computer laptop');
    const ids1 = res1.map((r) => r.id).sort();
    const ids2 = res2.map((r) => r.id).sort();
    expect(ids1).toEqual(ids2);
  });

  it('search with more specific query returns subset', async () => {
    const broad = await searchService.search('laptop');
    const specific = await searchService.search('laptop dell');
    const broadIds = new Set(broad.map((r) => r.id));
    const specificIds = specific.map((r) => r.id);
    // Every result from the specific query should be in the broad query
    specificIds.forEach((id) => {
      expect(broadIds.has(id)).toBe(true);
    });
  });
});

describe('PriceCalculator — metamorphic testing', () => {
  it('applying 10% discount then 10% surcharge != original (order matters)', () => {
    const original = 100;
    const discounted = applyDiscount(original, 10); // 90
    const surcharged = applySurcharge(discounted, 10); // 99
    expect(surcharged).not.toBe(original);
    // This is a *metamorphic relation*: the operations are not commutative
  });
});
```

### Cuándo usarlo

Para funciones sin oráculo claro: búsqueda, ranking, ML, transformaciones de datos,
algoritmos de recomendación. Define relaciones metamórficas que deben cumplirse
independientemente del input específico.

---

## 2.4 Differential testing

### Qué es

Compara contra una implementación de referencia o contra la versión anterior.

### Ejemplo de código

```typescript
describe('PaymentService — differential testing', () => {
  it('new fee calculator matches reference implementation', () => {
    fc.assert(
      fc.property(
        fc.record({
          amount: fc.float({ min: 0.01, max: 100000, noNaN: true }),
          currency: fc.constantFrom('USD', 'EUR', 'GBP', 'JPY'),
          paymentMethod: fc.constantFrom('card', 'bank_transfer', 'wallet'),
        }),
        (input) => {
          const newResult = newFeeCalculator(input);
          const referenceResult = referenceFeeCalculator(input);
          expect(newResult).toBeCloseTo(referenceResult, 2);
        }
      )
    );
  });

  it('new parser produces same output as old parser for all valid inputs', () => {
    const testInputs = loadTestInputs('fixtures/parser-inputs/');
    for (const input of testInputs) {
      const newOutput = newParser(input);
      const oldOutput = oldParser(input);
      expect(newOutput).toEqual(oldOutput);
    }
  });
});
```

### Cuándo usarlo

Cuando reescribes o refactorizas una implementación existente. Compara la nueva versión
contra la vieja para garantizar que no hay regresiones. También útil para comparar una
implementación rápida contra una referencia lenta pero correcta.

---

## 2.5 Fuzzing guiado por cobertura

### Qué es

Fuzzing que usa feedback de cobertura para guiar la generación de inputs hacia caminos no
explorados del código.

### Stack

- **C/C++**: [libFuzzer](https://llvm.org/docs/LibFuzzer.html) o [AFL](https://github.com/google/AFL)
- **Python**: [Atheris](https://github.com/google/atheris)
- **Node.js**: [jsfuzz](https://github.com/fuzzinglabs/jsfuzz) o [fast-check fuzz mode](https://github.com/dubzzz/fast-check)

### Ejemplo de código

```python
# Python con Atheris
import atheris

def test_parser_fuzz(data):
    fdp = atheris.FuzzedDataProvider(data)
    input_str = fdp.ConsumeUnicodeNoSurrogates(100)
    try:
        result = parse_config(input_str)
        assert result is not None
    except ValueError:
        pass  # Expected for invalid input
    except Exception as e:
        # Unexpected crash — fuzzing found a bug
        raise

atheris.Setup([], test_parser_fuzz)
atheris.Fuzz()
```

### Cuándo usarlo

Para parsers, deserializadores, y cualquier código que procesa input no confiable. El
fuzzing guiado por cobertura encuentra bugs que el property-based testing no alcanza
porque explora caminos específicos del código.

---

## 2.6 Grammar-based fuzzing

### Qué es

Genera inputs válidos (e inválidos) basándose en una gramática formal. Útil para lenguajes
DSL, queries, formatos estructurados.

### Stack

- **Python**: [fuzzingbook](https://github.com/uds-se/fuzzingbook) o [gramfuzz](https://github.com/fgsect/gramfuzz)
- **Node.js**: [fast-check con model-based](https://github.com/dubzzz/fast-check)

### Ejemplo de código

```python
from fuzzingbook.GrammarFuzzer import GrammarFuzzer

# Grammar for a simple query language
QUERY_GRAMMAR = {
    "<start>": ["<query>"],
    "<query>": ["<select> <from> <where>"],
    "<select>": ["SELECT *", "SELECT <fields>"],
    "<fields>": ["<field>", "<field>, <fields>"],
    "<field>": ["name", "email", "age", "status"],
    "<from>": ["FROM users", "FROM orders", "FROM products"],
    "<where>": ["WHERE <condition>", ""],
    "<condition>": ["<field> = <value>", "<field> > <value>"],
    "<value>": ["1", "0", "'test'", "'admin'"],
}

fuzzer = GrammarFuzzer(QUERY_GRAMMAR)

def test_query_parser_grammar_fuzz():
    for _ in range(1000):
        query = fuzzer.fuzz()
        try:
            result = parse_query(query)
            assert result is not None
        except SyntaxError:
            pass  # Invalid query — expected
        except Exception:
            raise  # Unexpected crash — bug found
```

### Cuándo usarlo

Para parsers de lenguajes específicos (SQL, JSON, XML, DSLs custom). La gramática define
qué es un input válido y el fuzzer genera variaciones que exploran edge cases sintácticos.

---

## 2.7 API fuzzing desde el esquema

### Qué es

Genera requests HTTP automáticamente desde un esquema OpenAPI/Swagger, explorando
combinaciones de parámetros, tipos, y valores edge.

### Stack

- **Python**: [Schemathesis](https://github.com/schemathesis/schemathesis)
- **REST**: [RESTler](https://github.com/microsoft/restler-fuzzer)

### Ejemplo de código

```bash
# Schemathesis — fuzzing desde OpenAPI spec
pip install schemathesis

# Run fuzzing against a running API
schemathesis run --base-url http://localhost:3000 openapi.yaml

# Con stateful testing (secuencias de requests)
schemathesis run --base-url http://localhost:3000 --stateful=links openapi.yaml
```

```python
# Python programmatic
import schemathesis

schema = schemathesis.openapi.from_url("http://localhost:3000/openapi.json")

@schema.parametrize()
def test_api(case):
    response = case.call()
    case.validate_response(response)
```

### Cuándo usarlo

Para APIs REST con especificación OpenAPI. Detecta crashes, 500s inesperados, y
comportamientos inconsistentes entre el esquema y la implementación.

---

## 2.8 Design by contract

### Qué es

Verificación de precondiciones, postcondiciones, e invariantes en runtime. Cada función
declara qué requiere (pre), qué garantiza (post), y qué mantiene (invariante).

### Stack

- **Node.js / TypeScript**: [@dciccale/contracts](https://github.com/dciccale/contracts) o custom decorators
- **Python**: [icontract](https://github.com/Parquery/icontract) o [PyContracts](https://github.com/AlexandreDecan/PyContracts)
- **Rust**: tipos del compilador + `debug_assert!`

### Ejemplo de código

```python
# Python con icontract
from icontract import require, ensure, invariant

@invariant(lambda self: self.balance >= 0, "balance must never be negative")
class BankAccount:
    def __init__(self, initial_balance: float):
        self.balance = initial_balance

    @require(lambda amount: amount > 0, "deposit amount must be positive")
    @ensure(lambda self, amount: self.balance == self.__old__.balance + amount)
    def deposit(self, amount: float):
        self.balance += amount

    @require(lambda amount: amount > 0, "withdrawal amount must be positive")
    @require(lambda self, amount: self.balance >= amount, "insufficient funds")
    @ensure(lambda self, amount: self.balance == self.__old__.balance - amount)
    def withdraw(self, amount: float):
        self.balance -= amount
```

```typescript
// TypeScript — custom contract decorator
function contract<T extends (...args: any[]) => any>(
  preconditions: ((...args: Parameters<T>) => void)[],
  postconditions: ((result: ReturnType<T>, ...args: Parameters<T>) => void)[]
) {
  return function (target: any, key: string, desc: PropertyDescriptor) {
    const original = desc.value;
    desc.value = function (...args: any[]) {
      preconditions.forEach((pre) => pre(...args));
      const result = original.apply(this, args);
      postconditions.forEach((post) => post(result, ...args));
      return result;
    };
  };
}

class PaymentService {
  @contract(
    [(amount: number) => { if (amount <= 0) throw new Error('amount must be positive'); }],
    [(result: any, amount: number) => { if (result.fee < 0) throw new Error('fee must be non-negative'); }]
  )
  processPayment(amount: number) {
    const fee = amount * 0.029;
    return { amount, fee, total: amount + fee };
  }
}
```

### Cuándo usarlo

Para APIs críticas donde las precondiciones y postcondiciones son parte del contrato.
Especialmente útil para servicios financieros, validadores, y cualquier función donde
el caller debe respetar reglas específicas.

---

## 2.9 Verificación formal (TLA+ / Alloy)

### Qué es

Modela el sistema a un nivel abstracto y usa verificación matemática para probar
propiedades (safety, liveness) exhaustivamente.

### Stack

- [TLA+](https://github.com/tlaplus/tlaplus) — usado por Amazon (DynamoDB, S3)
- [Alloy](https://github.com/AlloyTools/org.alloytools.alloy) — model checking ligero

### Ejemplo de código

```tla
---- MODULE OrderStateMachine ----
EXTENDS Naturals, Sequences

VARIABLES state

States == {"PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"}

Init == state = "PENDING"

Confirm == state = "PENDING" /\ state' = "CONFIRMED"
Ship    == state = "CONFIRMED" /\ state' = "SHIPPED"
Deliver == state = "SHIPPED" /\ state' = "DELIVERED"
Cancel  == (state = "PENDING" \/ state = "CONFIRMED") /\ state' = "CANCELLED"

Next == Confirm \/ Ship \/ Deliver \/ Cancel

Spec == Init /\ [][Next]_state

(* Safety: DELIVERED orders cannot transition to CANCELLED *)
CannotCancelDelivered == state = "DELIVERED" => state' # "CANCELLED"

(* Liveness: PENDING orders can always be confirmed or cancelled *)
AlwaysCancelable == state = "PENDING" => <> (state = "CONFIRMED" \/ state = "CANCELLED")
====
```

### Cuándo usarlo

Para algoritmos distribuidos, protocolos de consenso, sistemas con concurrencia, y
cualquier sistema donde un bug es catastrófico (infraestructura crítica). No es práctico
para aplicaciones CRUD normales.

---

## 2.10 Deterministic simulation testing

### Qué es

Simula el sistema completo con un scheduler determinista que controla el orden de
eventos, relojes, y fallos de red. Permite reproducir bugs que solo aparecen con
interleavings específicos.

### Stack

- **Rust**: [madsim](https://github.com/madsim-rs/madsim) o [Antithesis](https://antithesis.com/)
- **Go**: [Antithesis](https://antithesis.com/) o [dbxfer](https://github.com/cockroachdb/cockroach/tree/master/pkg/util)
- **Node.js**: [sinon fake timers](https://github.com/sinonjs/fake-timers) + custom scheduler

### Ejemplo de código

```typescript
import { install } from '@sinonjs/fake-timers';

describe('OrderService — deterministic simulation', () => {
  let clock: any;

  beforeEach(() => {
    clock = install({ now: new Date('2026-01-01T00:00:00Z') });
  });

  afterEach(() => clock.uninstall());

  it('order expires after 30 minutes if not confirmed', async () => {
    const order = await orderService.create({ items: [...] });
    expect(order.status).toBe('PENDING');

    // Advance time by 31 minutes
    clock.tick(31 * 60 * 1000);

    // Process expiration
    await orderService.processExpirations();

    const expired = await orderService.findById(order.id);
    expect(expired.status).toBe('EXPIRED');
  });

  it('retry with exponential backoff is deterministic', async () => {
    const attempts: number[] = [];
    let callCount = 0;

    const flakyService = async () => {
      callCount++;
      attempts.push(clock.now);
      if (callCount < 3) throw new Error('transient');
      return 'success';
    };

    const result = await retryWithBackoff(flakyService, {
      maxRetries: 5,
      baseDelay: 1000,
      maxDelay: 30000,
    });

    expect(result).toBe('success');
    expect(attempts).toHaveLength(3);
    // Verify backoff intervals: 1s, 2s
    expect(attempts[1] - attempts[0]).toBe(1000);
    expect(attempts[2] - attempts[1]).toBe(2000);
  });
});
```

### Cuándo usarlo

Para sistemas distribuidos, sistemas con timeouts y retries, y cualquier código que
depende del orden de eventos. Especialmente útil para detectar race conditions y
deadlocks.

---

## Skills y GitHub Projects

### BMad Skills relevantes

- **`bmad-review-adversarial-general`** — revisión cínica de las propiedades definidas.
  Útil para encontrar propiedades que parecen correctas pero tienen falsos positivos o
  que no capturan los invariantes reales del sistema.
- **`bmad-eval-runner`** — ejecuta eval sets en entornos aislados. Útil para verificar
  que los property-based tests son reproducibles y deterministas.

### GitHub Projects y npm packages

| Tool | Repo / Package | Instalación |
|------|----------------|-------------|
| fast-check | [dubzzz/fast-check](https://github.com/dubzzz/fast-check) | `npm i -D fast-check` |
| Hypothesis | [HypothesisWorks/hypothesis](https://github.com/HypothesisWorks/hypothesis) | `pip install hypothesis` |
| proptest (Rust) | [AltSysrq/proptest](https://github.com/AltSysrq/proptest) | `cargo add proptest` |
| Schemathesis | [schemathesis/schemathesis](https://github.com/schemathesis/schemathesis) | `pip install schemathesis` |
| Atheris | [google/atheris](https://github.com/google/atheris) | `pip install atheris` |
| icontract | [Parquery/icontract](https://github.com/Parquery/icontract) | `pip install icontract` |
| TLA+ | [tlaplus/tlaplus](https://github.com/tlaplus/tlaplus) | Descargar de [lamport.org](https://lamport.org/tla/tla.html) |
| Alloy | [AlloyTools](https://github.com/AlloyTools/org.alloytools.alloy) | Descargar de [alloytools.org](https://alloytools.org/) |
| madsim (Rust) | [madsim-rs/madsim](https://github.com/madsim-rs/madsim) | `cargo add madsim` |
| fuzzingbook | [uds-se/fuzzingbook](https://github.com/uds-se/fuzzingbook) | `pip install fuzzingbook` |
