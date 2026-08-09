# Use Cases

**Concept.** A use-case represents one application operation. It coordinates
existence, invariants, persistence, integrations and events without knowing
about HTTP, queues or the runtime.

## Invariants

| ID | Law |
|---|---|
| UC-1 | One operation = one class = one public `execute(input)` method; queries are use-cases too. |
| UC-2 | The input shape is validated at the boundary; the use-case does not repeat Zod nor validate format. |
| UC-3 | Existence/ownership are checked before the rule; absence produces the domain's typed error. |
| UC-4 | A business rule is protected by an entity guard/mutation, not by a duplicated `if` in the use-case. |
| UC-6 | The return is an entity, a collection/page of entities or `void`; never a database row or an HTTP DTO. |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-LAY-4` | Cross-domain through the public operation. |
| `ARC-CON-1` | Every write declares its strategy. |
| `ARC-CON-5` | Event only after the write is confirmed. |
| `ARC-OBS-4` | Error logged once at the boundary. |
| `ARC-TOP-3` | The app registers only the closure it consumes. |


## Validation ladder

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

## Allowed dependencies

- The `DatabaseService` typed repository for trivial persistence.
- Its own repository when it adds policy/composition.
- Another domain's public use-case.
- A service offered directly by a Turystack lib.
- A local adapter for an integration not covered by the libs.
- A publisher for events after success.

Do not inject the request, the response, a transport decorator or another
domain's repository.

## Consistency choice

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

Do not hold a database transaction open during external I/O.

## Example

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

## Registration per consumer

The API/handler root module registers the use-case and its transitive dependencies.
Do not create `OrderModule` just to re-export all of the domain's providers. A
small handler must not carry operations it never runs.

## Never do

- Create an `OrderService` with several business methods.
- Validate email, enum or required again inside `execute`.
- Implement a guard with an `if` over state instead of calling the entity.
- Access another domain's repository.
- Return an invented partial object or an ORM row.
- Call a provider before persisting without a consistency strategy.
- Catch an error only to `logger.error` + `throw`; the boundary already logs the
  failure.
- Swallow an error or return `null` after a failure.
