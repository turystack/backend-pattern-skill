# Testing

**Concept.** Three levels by the nature of the test: **unit** (pure functions), **integration** (use-cases/repositories with **everything mocked**), **e2e** (controllers/schedulers/consumers **with no mocks**). A test documents behavior and locks down regressions; test data comes from a factory, not from literals.

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Drizzle · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `TST-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `TST-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**TST-1 — three-level pyramid.**

Each level has a different nature; they are not interchangeable:

- **Unit** — a **simple/pure** function, with no injected dependencies and no mocks: a utility, an entity guard/reader, a mapper. Tests input → output.
- **Integration** — covers **use-cases and repositories** with **all** dependencies mocked (no real IO). Validates the `execute` flow (existence, guards, error propagation) in total isolation from database/network.
- **E2E** — **no mocks at all**, exercising the entry points end-to-end (app + real test infra): **controllers** (via HTTP request), **schedulers** and **consumers** (fire the trigger, check the persisted effect).

**[TST-1]**

**TST-2 — test data via factory, never literals.**

Test data comes from a factory derived from the schema (`makeXxx`). Never build an entity by hand with all its fields; do not use literals scattered across the tests. The factory guarantees that any change to the schema's shape is felt in a single place. **[TST-2]**

**TST-3 — shared setup, never duplicated.**

Common setup and teardown (logger spy, database reset, base state seed) live in a centralized setup file. Never replicate a spy or a seed in the body of an individual spec. **[TST-3]**

**TST-4 — multiple inputs → table of named cases.**

Several inputs for the same behavior → a table of named cases; never copy the same test block per value variation. **[TST-4]**

**TST-5 — integration: all IO mocked.**

At the integration level, **every** access to an external resource (database, messaging, third-party HTTP) is replaced by a mock. No integration test opens a real connection, writes to a database or makes a network request. **[TST-5]**

**TST-6 — e2e: no mocks.**

At the e2e level, **no** dependency is replaced by a mock. The test exercises the full stack (route → controller → use-case → repository → database). **[TST-6]**

**TST-7 — controller e2e covers auth/ACL + route prefix + round-trip.**

A controller e2e validates: authentication (Bearer token), authorization (the route's permission), the surface's correct prefix (`/api/v1/ops/`, `/api/v1/bko/`, etc.) and the minimal round-trip (create → get confirms the state). **[TST-7]**

**TST-8 — schedulers and consumers have their own e2e.**

Schedulers and consumers are tested at the e2e level: fire the trigger directly (no HTTP), check the side effect persisted in the database. **[TST-8]**

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Class | Detector (🛠️) |
|---|---|---|---|
| TST-2 | Test data via a factory derived from the schema; never a literal entity | constitutional | Unit/Int/E2E scenarios / ❌ |
| TST-3 | Centralized shared setup; never duplicated per spec | constitutional | ❌ |
| TST-4 | Multiple inputs → table of named cases; never copy a test block per value | constitutional | Int scenario / ❌ |
| TST-5 | Integration: all IO mocked; no real database/network access | constitutional | Int scenario / ❌ |
| TST-6 | E2E: no mocks; full stack | constitutional | E2E scenario / ❌ |
| TST-7 | Controller e2e: auth + ACL + prefix + round-trip | constitutional | E2E scenario / ❌ |
| TST-8 | Schedulers/consumers have their own e2e (trigger → effect in the database) | constitutional | E2E scenario |
| TST-L1 | Suffixes: `*.test.ts` placed next to the file it tests (use-case: `use-cases/<op>/<op>.test.ts`); `*.e2e.test.ts` run by the separate config `vitest.e2e.config.ts` | stack lint | Unit/Int/E2E scenarios |
| TST-L2 | Mocks via Vitest (`vi.fn()`, `Mocked<T>`, `mockResolvedValueOnce`, `mockRejectedValueOnce`) | stack lint | Int scenario |
| TST-L3 | NestJS TestingModule: `DatabaseService` mocked per table; adapters mocked per interface/token; cross-domain per class | stack lint | Int scenario |
| TST-L4 | Controller e2e via supertest (`Test.createTestingModule` + `AppModule`) | stack lint | E2E scenario |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-TST-1` | Each level asks a distinct question. |
| `ARC-TST-2` | The lowest level runs without infra. |
| `ARC-TST-3` | Data comes from a factory derived from the contract. |
| `ARC-TST-4` | Infra semantics are proven against real infra. |
| `ARC-TST-5` | Handlers have a duplicate and out-of-order case. |
| `ARC-TST-6` | Tests with infra live in a separate config. |


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · Vitest):**

- **TST-1** — unit and integration in `*.test.ts` **placed next to the file it tests** (a use-case's spec lives in the operation's folder: `use-cases/<op>/<op>.test.ts`); e2e in `*.e2e.test.ts`, run by the separate config `vitest.e2e.config.ts`.
- **TST-2** — factory `makeOrder` derived from the schema in `order.mock.ts` (a `<domain>.mock.ts` file next to the domain); override fields via spread.
- **TST-3** — global setup in `vitest.setup.ts`; referenced in `vitest.config.ts` (and in the e2e config).
- **TST-4** — `it.each([...])('description $field', ({ ... }) => { ... })`.
- **TST-5** — `Test.createTestingModule` with `DatabaseService` and adapters mocked; no real database/HTTP connection.
- **TST-6** — `Test.createTestingModule({ imports: [AppModule] })` with no overrides at all; real test database.
- **TST-7** — `supertest` with the `Authorization: Bearer <token>` header; assert on the status code + prefix.
- **TST-8** — call the job/consumer's `execute` directly; check via `databaseService.orders.count()` or `findFirst`.
- **TST-L1** — suffixes `.test.ts` / `.e2e.test.ts` configured in `vitest.config.ts` / `vitest.e2e.config.ts`; coverage with thresholds of **90** (lines/functions/branches/statements) in `vitest.config.ts`.
- **TST-L2** — Vitest: `vi.fn()`, `Mocked<T>`, `mockResolvedValueOnce`, `mockRejectedValueOnce`.
- **TST-L3** — database via `{ provide: DatabaseService, useValue: databaseServiceMock }` (mocked per table); adapter per interface/token `{ provide: BILLING_ADAPTER, useValue: billingAdapterMock }`; cross-domain per class `{ provide: ProcessPaymentUseCase, useValue: mock }`.
- **TST-L4** — `import request from 'supertest'`.

### ✅ How to do it

**Unit — pure function (utility, entity guard): no DI, no mocks:** `[TST-1, TST-2, TST-4, TST-L1]`

```typescript
import { describe, expect, it } from 'vitest'

import { maskEmail } from '@/domains/auth/mask-email'

import { exceptions } from '@/exceptions'

import { makeOrder } from './order.mock'

describe('maskEmail', () => {
  it.each([
    { expected: 'j***@doe.com', input: 'john@doe.com' },
    { expected: 'a***@b.com', input: 'a@b.com' },
  ])('masks $input', ({ input, expected }) => {
    expect(maskEmail(input)).toBe(expected)
  })
})

describe('OrderEntity.checkIfCanPay', () => {
  it('throws when already paid', () => {
    expect(() => makeOrder({ status: 'PAID' }).checkIfCanPay()).toThrow(
      exceptions.order.alreadyPaid,
    )
  })
})
```

**Integration — use-case with everything mocked (`DatabaseService` per table); one spec per use-case, placed in the operation's folder (`use-cases/create-order/create-order.test.ts`):** `[TST-1, TST-2, TST-4, TST-5, TST-L1, TST-L2, TST-L3]`

```typescript
import { Test } from '@nestjs/testing'
import { beforeEach, describe, expect, it, type Mocked, vi } from 'vitest'

import { DatabaseService } from '@turystack/nestjs-database'

import { exceptions } from '@/exceptions'

import { makeOrder } from '../../order.mock'
import { CreateOrderUseCase } from './create-order'

describe('CreateOrderUseCase', () => {
  let orderRepository: Mocked<DatabaseService['orders']>
  let createOrderUseCase: CreateOrderUseCase

  beforeEach(async () => {
    orderRepository = {
      create: vi.fn(),
      findById: vi.fn(),
      findMany: vi.fn(),
      updateById: vi.fn(),
    } as unknown as Mocked<DatabaseService['orders']>

    const moduleRef = await Test.createTestingModule({
      providers: [
        CreateOrderUseCase,
        { provide: DatabaseService, useValue: { orders: orderRepository } },
      ],
    }).compile()

    createOrderUseCase = moduleRef.get(CreateOrderUseCase)
  })

  it('saves and returns the order', async () => {
    const order = makeOrder()
    orderRepository.create.mockResolvedValueOnce(order)

    await expect(
      createOrderUseCase.execute({ customerId: order.customerId, totalInCents: 9990 }),
    ).resolves.toBe(order)
  })

  // multiple inputs → it.each is mandatory
  it.each([
    { reason: 'zero', totalInCents: 0 },
    { reason: 'negative', totalInCents: -1 },
  ])('rejects invalid total ($reason)', async ({ totalInCents }) => {
    await expect(
      createOrderUseCase.execute({ customerId: 'customer', totalInCents }),
    ).rejects.toThrow(exceptions.order.invalidTotal)
  })
})
```

> The factory stays in `order.mock.ts`, at the domain's root — shared by every spec. Each operation has its own spec, cast from the same mold:

```typescript
// use-cases/get-order/get-order.test.ts
it('returns the order when it exists', async () => {
  const order = makeOrder()
  orderRepository.findById.mockResolvedValueOnce(order)

  await expect(getOrderUseCase.execute(order.id)).resolves.toBe(order)
})

it('throws notFound when missing', async () => {
  orderRepository.findById.mockResolvedValueOnce(undefined)

  await expect(getOrderUseCase.execute('missing')).rejects.toThrow(
    exceptions.order.notFound,
  )
})

// use-cases/delete-order/delete-order.test.ts
it('soft-deletes via updateById', async () => {
  const order = makeOrder()
  orderRepository.findById.mockResolvedValueOnce(order)
  orderRepository.updateById.mockResolvedValueOnce(order)

  await deleteOrderUseCase.execute(order.id)

  expect(orderRepository.updateById).toHaveBeenCalledWith(
    order.id,
    expect.objectContaining({ deletedAt: expect.any(Date) }),
  )
})
```

**Integration — adapter mocked per interface (never the real SDK):** `[TST-1, TST-5, TST-L1, TST-L2, TST-L3]`

```typescript
// adapter mocked via the interface + token; no real SDK/HTTP
const billingAdapter = { chargeInvoice: vi.fn() } as Mocked<IBillingAdapter>
billingAdapter.chargeInvoice.mockResolvedValueOnce({ status: 'PAID' })
// in the TestingModule: { provide: BILLING_ADAPTER, useValue: billingAdapter }
```

**E2E — controller: full app + supertest; auth/ACL + prefix + round-trip:** `[TST-1, TST-6, TST-7, TST-L1, TST-L4]`

```typescript
import { Test } from '@nestjs/testing'
import request from 'supertest'

describe('Orders (e2e)', () => {
  let app: INestApplication

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile()
    app = moduleRef.createNestApplication()
    await app.init()
  })

  afterAll(async () => {
    await app.close()
  })

  it('POST then GET returns the created order', async () => {
    const created = await request(app.getHttpServer())
      .post('/api/orders')
      .set('Authorization', `Bearer ${accessToken}`)
      .send({ customerId, totalInCents: 9990 })
      .expect(201)

    await request(app.getHttpServer())
      .get(`/api/orders/${created.body.id}`)
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(200)
  })
})
```

**E2E — scheduler/consumer: fire the trigger directly, check the effect in the database:** `[TST-1, TST-6, TST-8, TST-L1]`

```typescript
// scheduler/consumer e2e: fire the trigger and check the effect (no mocks)
await processOverdueInvoicesUseCase.execute()
expect(await databaseService.invoices.count()).toBeGreaterThan(0)
```

### ❌ Never do

```typescript
// ❌ [TST-4] copying the it per value instead of it.each (multiple inputs)
it('rejects 0', () => {})
it('rejects -1', () => {})

// ❌ [TST-2] Map store / hidden seed routing by id
const store = new Map(); orderRepository.findById = (id) => store.get(id)

// ❌ [TST-2] building the entity by hand instead of using the factory
const order = new OrderEntity({ id: '1', /* ...all the fields */ })

// ❌ [TST-3] duplicating the logger spy / seed reset in the spec (it comes from the shared setup)
beforeEach(() => vi.spyOn(LoggerService.prototype, 'log'))

// ❌ [TST-5] real IO in integration (EVERYTHING is mocked)
await databaseService.orders.create(data)

// ❌ [TST-6] a mock in e2e (e2e has NO mocks — it calls the real entry point)
orderRepository.findById = vi.fn()

// ❌ [TST-7] e2e without isolating state between tests / hitting a database that is not a test one
```

> ⚠️ During feature delivery, domain tests are **deferred** — validate with `typecheck` + `biome`; the standards above apply once the tests are written.
