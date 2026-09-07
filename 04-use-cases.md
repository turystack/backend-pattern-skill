# Use Cases

**Concept.** A use-case represents one application operation. It coordinates
existence, invariants, persistence, integrations and events without knowing
about HTTP, queues or the runtime.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack. The split, `XXX-n` versus `XXX-Ln`, and why an `ARC-…` law is
> cited and never restated: `turystack-backend-pattern` › *How a section is
> written*.

---

**Rules defined here:** `UC-1` · `UC-2` · `UC-3` · `UC-4` · `UC-6` — the law
is the *Invariants* table below; every ❌ item cites the id it violates.

**Retired ids:** `UC-5` — retired, not renumbered. A review or commit citing
one points at a rule that no longer exists; the number is never reused.

## 🌐 Generic pattern (portable — stack-independent)

### Invariants (the law the gates enforce)

| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| UC-1 | One operation = one class = one public `execute(input)`; a query is a use-case too | constitutional | `gate:use-case-shape` | Example / ❌ |
| UC-2 | Input shape is validated at the boundary; the use-case never revalidates format | constitutional | `manual` | Validation ladder / ❌ |
| UC-3 | Existence and ownership are proven before the rule; absence raises the domain's typed error | constitutional | `manual` | Validation ladder / ❌ |
| UC-4 | A business rule is protected by an entity guard/mutation, never by an `if` copied into the use-case | constitutional | `gate:no-rule-in-use-case` | Example / ❌ |
| UC-6 | The return is an entity, a collection/page of entities or nothing — never a database row or a transport DTO | constitutional | `manual` | Example / ❌ |

Ids are stable across versions; a gap is a law that moved to the constitution.

**Why UC-1 and not a service with many methods.** A class per operation is what
makes the dependency list of that operation visible. An `OrderService` with
eight methods carries the union of eight dependency sets, so a handler that runs
one of them is forced to register all eight (`ARC-TOP-3`), and no reader can
tell which of the eight actually uses the publisher.

**Why UC-6 refuses to return a row.** A database row is the persistence format;
handing it to a caller makes the storage schema part of the operation's
contract, and every consumer becomes a reason not to change a column.

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-LAY-4` | Cross-module access only through the public operation. | inject another domain's use-case, never its repository |
| `ARC-CON-1` | Every write declares its strategy. | `@Transactional()`, explicit compensation, or persist → publish |
| `ARC-CON-3` | A transaction is never held open across an external call. | external I/O runs outside `@Transactional()` |
| `ARC-CON-5` | An event is emitted only after the write commits. | `publisher.publish` after the persistence call returns |
| `ARC-OBS-4` | An error is logged once, at the highest boundary. | the use-case rethrows; the filter logs |
| `ARC-TOP-3` | An app registers only the closure it consumes. | the handler registers this use-case, not the domain's whole provider set |

### Validation ladder

Inside `execute`, preserve the order:

1. fetch by PK/FK and prove existence;
2. prove the ownership/scope that IAM did not resolve;
3. call the entity's guards and mutations;
4. persist;
5. run the integration or publish the event according to the strategy;
6. return the canonical result.

Structural validation (required, enum, format, length) belongs to the
controller/handler schema. The HTTP category is a server-side translation; the
use-case throws typed application errors.

### Consistency choice

```text
one write
└── normal operation

multiple writes to the same database
└── declarative transaction in the use-case

database + reversible external effect
└── explicit saga/compensation

non-reversible, asynchronous or retry-prone external effect
└── persist state → publish event → idempotent handler runs the effect
```

The saga belongs to the use-case flow, not to a mandatory global folder.
Extract a coordinator only when the sequence is reused or complex enough to own
its own lifecycle.

When using compensation:

1. persist the source of truth;
2. record how to undo it immediately;
3. run the external effect;
4. persist the external identifier/result;
5. on failure, compensate and rethrow.

Do not hold a database transaction open during external I/O (`ARC-CON-3`).

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack)

### Allowed dependencies

- The `DatabaseService` typed repository for trivial persistence.
- Its own repository when it adds policy/composition.
- Another domain's public use-case.
- A service offered directly by a Turystack lib.
- A local adapter for an integration not covered by the libs.
- A publisher for events after success.

Do not inject the request, the response, a transport decorator or another
domain's repository.

A compensating sequence that outgrows the use-case has an owner:
`@turystack/saga`. Reach for it before hand-rolling a coordinator
(`ARC-LAY-8`).

### Example

```typescript
export type CancelOrderInput = {
  orderId: Order['orderId']
  organizationId: Organization['organizationId']
}

@Injectable()
export class CancelOrderUseCase {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly publisher: PublisherService,
  ) {}

  @Transactional()
  async execute(input: CancelOrderInput) {
    const order = await this.orderRepository.findById(input.orderId)

    order.checkOrganization(input.organizationId)
    order.cancel()

    const updated = await this.orderRepository.updateById(
      order.orderId,
      order,
    )

    this.publisher.publish({
      data: updated,
      destination: 'TOPIC',
      name: 'order.cancelled',
    })

    return updated
  }
}
```

- The input derives its types from the canonical schemas/entities.
- In the TypeScript stack, let the return of `execute` be inferred; do not create
  a duplicate `CreateOrderOutput`.
- `publish` and `@Transactional` follow the API documented by their respective libs.

### Registration per consumer

The API/handler root module registers the use-case and its transitive dependencies.
Do not create `OrderModule` just to re-export all of the domain's providers. A
small handler must not carry operations it never runs.

### ❌ Never do

- `[UC-1]` Create an `OrderService` with several business methods.
- `[UC-2]` Validate email, enum or required again inside `execute`.
- `[UC-4]` Implement a guard with an `if` over state instead of calling the entity.
- `[ARC-LAY-4]` Access another domain's repository.
- `[UC-6]` Return an invented partial object or an ORM row.
- `[ARC-CON-1]` Call a provider before persisting without a consistency strategy.
- `[ARC-CON-3]` Keep `@Transactional()` open around an external call.
- `[ARC-OBS-4]` Catch an error only to `logger.error` + `throw`; the boundary already logs the failure.
- `[ARC-ERR-7]` Swallow an error or return `null` after a failure.
