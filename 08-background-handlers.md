# Background Handlers

**Concept.** A consumer, a queue handler and a scheduled job are delivery
boundaries, just like a controller. They translate the envelope they receive,
validate the input, delegate to a use-case and let the infrastructure decide
retry and dead-letter.

## Invariants

| ID | Law |
|---|---|
| BGH-2 | The schema validates the payload at the boundary before any effect. |
| BGH-6 | A schedule fires one operation; calendar and timezone rules are validated app configuration. |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-6` | Delivery boundary is thin. |
| `ARC-DEL-3` | Scheduler discovers and distributes. |
| `ARC-DEL-4` | One item's failure does not stop the others. |
| `ARC-DEL-5` | Failure rethrown to the infra. |
| `ARC-DEL-6` | Batch with limited concurrency. |
| `ARC-DEL-7` | Overlapping execution is a decision. |
| `ARC-IDM-1` | Duplicate delivery is assumed. |
| `ARC-CON-5` | Event only after success. |


## Contexts

### Standalone

- `@Subscriber` reacts to an event inside the process.
- `@Schedule` fires a scheduled operation inside the API.
- The handler sits next to the domain it delivers to and calls the same use-case
  that a route would call.

### Monorepo or serverless

- An app handler is an isolated delivery point and uses `@Handler`.
- The app imports operations from `@repo/domains` and registers only the provider
  closure that handler needs.
- Each app owns its `config.schema.ts`; it does not read `process.env` inside the
  handler.

The decorators, supported adapters, envelopes and retry options belong to the
`@turystack/nestjs-serverless` documentation.

## Shape example

```typescript
@Handler({ schema: paymentRequestedSchema })
export class ProcessPaymentHandler {
  constructor(
    private readonly processPaymentUseCase: ProcessPaymentUseCase,
  ) {}

  async execute(input: PaymentRequested) {
    return this.processPaymentUseCase.execute(input)
  }
}
```

## Never do

- Copying a use-case rule into the handler.
- Accessing `DatabaseService` or a repository directly.
- Catching an error and returning success/null.
- Running an unbounded `Promise.all` over a dynamic batch.
- Inventing a wrapper around a queue, schedule or logger the lib already provides.
