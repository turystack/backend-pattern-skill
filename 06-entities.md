# Entities

**Concept.** Rich domain model: state **and** behavior (rules, transitions, invariants), not an anemic bag of data. The entity is the **owner of the business invariants** — it is what the use-case calls at **step 3 of the ladder** (the guards that throw 409). Guard before mutating; invalid state is unreachable. The entity **never** knows about infra (repository, adapter, ORM, HTTP).

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Drizzle · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `ENT-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `ENT-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**ENT-1 — the entity owns the invariants.** The entity is the **only** one responsible for deciding business rules: the use-case delegates via guards — it never decides the 409 inline. A `checkIfX()`/`checkIfCanX()` guard covers **one** invariant, reuses a reader and throws the typed exception of the category (typically conflict 409; may be 401/403 in auth). **[ENT-1]**

**ENT-2 — guards without IO.** A guard does no IO (no repo, no adapter, no external I/O): it decides using only its own state and the arguments it receives. A rule that needs the database lives in the use-case, not in the entity. **[ENT-2]**

**ENT-3 — mutation calls guard(s) before mutating.** Every method that transitions state (mutation) calls the relevant guard(s) **before** changing any field. Invalid state is unreachable — the rule enforces that structurally. **[ENT-3]**

**ARC-LAY-2 — the entity never knows infra.** The entity does not import or reference a repository, adapter, ORM, database or HTTP. Its scope is the pure domain model. **[ARC-LAY-2]**

**ENT-5 — field ordering.** Always in this order (skip groups that do not exist):

1. **PK** — identifier of the aggregate.
2. **FKs** — and, for each FK, the **hydrated join next to it** is the **Identity** of the related domain (`customerId` → `customer: CustomerIdentity`), never the full entity.
3. **Important fields** — the core of the business.
4. **Less important fields** — secondary/optional.
5. **Booleans** — flags.
6. **Status** — state enums.
7. **Timestamps + Audit** — `createdAt`/`updatedAt`/`deletedAt`, then `createdBy`/`updatedBy`/`deletedBy`.

**[ENT-5]**

**ARC-CTR-3 — FK hydrated as Identity.** Every FK always comes with the hydrated join as the **Identity** of the related domain (key fields only), never the full entity, never a bare id on its own. **[ARC-CTR-3]**

**ENT-7 — every `if` with a `{}` block.** No `if` without braces, no ternary hiding a `throw`. One statement per line. Applies to guards, readers and mutations. **[ENT-7]**

**ENT-8 — readers derive state, with no side effects.** Readers (`isX()`, `hasX()`) compute and return derived state; they never mutate a field, never do IO. **[ENT-8]**

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Type | Detector (🛠️) |
|---|---|---|---|
| ENT-1 | Entity decides the rule; use-case delegates via guard; never a 409 inline in the use-case | constitutional | Scenario 1 / ❌ |
| ENT-2 | Guard covers one invariant; no IO (no repo/adapter/external I/O) | constitutional | Scenario 1 / ❌ |
| ENT-3 | Mutation calls guard(s) before mutating a field | constitutional | Scenario 1 / ❌ |
| ENT-5 | Ordering: PK → FK+Identity → important → less important → booleans → status → timestamps+audit | constitutional | Scenario 2 / ❌ |
| ENT-7 | Every `if` with a `{}` block; no `if` without braces; no ternary hiding a throw | constitutional | Scenario 1 / ❌ |
| ENT-8 | Readers derive state, with no side effects and no IO | constitutional | Scenario 1 |
| ENT-L1 | Class decorated with `@Entity()` | stack lint | Scenarios 1–2 |
| ENT-L2 | Hydration in the constructor via `Object.assign(this, data)` | stack lint | Scenarios 1–2 / ❌ |
| ENT-L3 | Fields typed by indexed access on the schema (`field!: Type['field']`); never by hand | stack lint | Scenarios 1–2 / ❌ |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-LAY-2` | Domain does not import infra. |
| `ARC-CTR-3` | FK referenced by Identity. |


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into real code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · @turystack):**

- **ENT-1** — `entity.checkIfX()` throws the typed exception from `@turystack/exceptions` (conflict category → 409; `unauthorized`/`forbidden` in auth) — see `04-use-cases` (UC-3), `10-error-handling`.
- **ENT-2** — a guard is a synchronous method, no `async`, no `@Inject`; it takes only primitives or types from its own domain.
- **ENT-3** — a mutation is a `void` method that calls `this.checkIfX()` on the first line, before any field assignment.
- **ARC-LAY-2** — no import of `@turystack/nestjs-database`, `DatabaseService`, `Repository`, `Adapter` or `HttpService` in the entity.
- **ENT-5** — class convention enforced in review (see `00-overview`). The semantic order is possible because `@turystack/backend-config` turns `useSortedKeys` off for `**/*.entity.ts` — in every other file Biome sorts alphabetically.
- **ARC-CTR-3** — Identity via the repository's `with` config: `{ user: { columns: { userId: true, name: true } } }`; the `XxxIdentity` type is inferred from the schema via `Pick` (see `05-repositories`, REP-5).
- **ENT-7** — Biome (via `@turystack/backend-config`) enforces mandatory braces; review checks for a `{}` block in every generated guard.
- **ENT-8** — a reader returns `boolean` (or a derived primitive); it never modifies `this`.
- **ENT-L1** — `@Entity()` decorator from `@turystack/entity` (registers the class with superjson — the entity survives event/payload serialization).
- **ENT-L2** — `Object.assign(this, data)` in the constructor; TS infers the types via definite assignment (`!`).
- **ENT-L3** — `field!: Type['field']` — the type comes from the inferred Zod schema; if the schema changes, the field changes with it.
- **PKs** — the entity **never** generates an id: the uuidv7 PK is auto-generated by the database lib in the typed repo's `create` (see `05-repositories`, `05-repositories`).

### ✅ How to do it

**Scenario 1 — full anatomy: static, readers, guards, mutations:** `[ENT-1, ENT-2, ENT-3, ENT-7, ENT-8, ENT-L1, ENT-L2, ENT-L3]`
```typescript
@Entity()
export class InvoiceEntity {
  static MAX_OPEN_PER_USER = 5

  invoiceId!: Invoice['invoiceId']
  totalInCents!: Invoice['totalInCents']
  status!: Invoice['status']

  constructor(data: Invoice) {
    Object.assign(this, data)
  }

  // readers — derive state, no side effects (ENT-8)
  isOpen(): boolean {
    return this.status === 'OPEN'
  }

  hasBalance(): boolean {
    return this.totalInCents > 0
  }

  // guards — one invariant each; always with braces {} (ENT-1, ENT-2, ENT-7)
  checkIfCanPay(): void {
    if (!this.isOpen()) {
      throw new exceptions.invoice.alreadyPaid()
    }
  }

  checkIfHasBalance(): void {
    if (!this.hasBalance()) {
      throw new exceptions.invoice.emptyBalance()
    }
  }

  // mutation — calls guard(s) before mutating (ENT-3)
  markAsPaid(): void {
    this.checkIfCanPay()
    this.checkIfHasBalance()
    this.status = 'PAID'
  }
}
```

**Scenario 2 — field ordering and FKs with Identity next to them:** `[ENT-5, ARC-CTR-3, ENT-L1, ENT-L2, ENT-L3]`
```typescript
@Entity()
export class OrderEntity {
  // 1. PK
  orderId!: Order['orderId']

  // 2. FK + join next to it = Identity of the related domain (slim)
  customerId!: Order['customerId']
  customer!: Order['customer'] // CustomerIdentity
  paymentId!: Order['paymentId']
  payment!: Order['payment'] // PaymentIdentity

  // 3. important fields
  totalInCents!: Order['totalInCents']
  items!: Order['items'] // main join (1:N) = full

  // 4. less important fields
  note!: Order['note']

  // 5. booleans
  isGift!: Order['isGift']

  // 6. status
  status!: Order['status']

  // 7. timestamps + audit
  createdAt!: Order['createdAt']
  updatedAt!: Order['updatedAt']
  deletedAt!: Order['deletedAt']
  createdBy!: Order['createdBy']
  updatedBy!: Order['updatedBy']

  constructor(data: Order) {
    Object.assign(this, data)
  }
}
```

### ❌ Never do

```typescript
// ❌ [ENT-7] if without braces — ALWAYS use a {} block
checkIfCanPay(): void {
  if (!this.isOpen()) throw new exceptions.invoice.alreadyPaid()
}

// ❌ [ENT-2] guard doing IO (a rule that needs the database lives in the use-case, not in the entity)
async checkIfNumberIsUnique() {
  if (await this.invoiceRepository.existsByNumber(this.number)) {
    throw new exceptions.invoice.numberAlreadyExists()
  }
}

// ❌ [ARC-LAY-2] entity importing infra
import { DatabaseService } from '@turystack/nestjs-database'

// ❌ [ENT-L3, ENT-L2] typing the field by hand / hydrating field by field
status: 'OPEN' | 'PAID'
constructor(data: Invoice) { this.status = data.status }

// ❌ [ENT-3] mutating without a guard
markAsPaid() { this.status = 'PAID' }

// ❌ [ENT-5, ARC-CTR-3] fields out of order (timestamp/status in the middle, FK far from its join)
orderId!: Order['orderId']
createdAt!: Order['createdAt']
customerId!: Order['customerId']
status!: Order['status']
customer!: Order['customer']
```
