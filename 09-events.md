# Events

**Concept.** Event-driven: the emitter announces that **something happened** (a fact in the past) and does not know who reacts; reactors subscribe. It decouples the side effect from the main flow. In-process (synchronous to the process) or distributed (broker).

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Drizzle · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `EVT-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `EVT-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**EVT-1 — event naming.**

Name in the `<domain>.<action>` format in the **past** — it announces an accomplished fact, not an intention. Examples: `invite.created`, `trip.materialized`, `order.cancelled`. **[EVT-1]**

**ARC-CON-5 — emit only after success; never before.**

The event is emitted **after** the main operation finishes successfully (persist, computation, etc.). Emitting before success is a false positive: reactors react to something that may not have happened. **[ARC-CON-5]**

**EVT-3 — the use-case is the publisher; every mutation emits.**

Events are the application layer's responsibility: the publisher is the **use-case**, always fire-and-forget — **never** `await`. Controllers, repositories and others do not emit. Every use-case that **mutates** state or performs a **business action** (`create`/`update`/`delete`/`accept`/`cancel`/`markAs`…) **must** emit an event after success — by default, it is a rule, not an option. A read use-case (`get`/`list`) does **not** emit. **[EVT-3]**

**EVT-4 — a side effect is a reaction, never a direct call.**

The emitter neither injects nor calls the service responsible for the side effect (notification, audit, push). Injecting `NotificationService` inside a use-case couples the two — any change to the notification propagates into the domain. The side effect **reacts** to the event; the emitter does not know who reacts. **[EVT-4]**

**EVT-5 — reactor named after the intent, not after the event.**

The handler is not called `onInviteCreated` (the event's name) — it is called `sendInviteEmail` (the intent of what it does). Naming by event creates "do-everything" handlers that pile up responsibilities. Naming by intent forces one handler per responsibility. **[EVT-5]**

**EVT-6 — payload: full entity in the simple case; normalized at high volume.**

Two forms; **always question which one to use**:

- **Simple case (default)** → emit the **full entity**; **zero-IO** handler (everything is in the payload, no extra query).
- **High volume** → **normalize** the payload before sending (ids/keys only) to keep the bus light; the handler loads whatever it needs.

The handler **never** runs an extra query when it could have received the data in the payload (simple case). **[EVT-6]**

**EVT-7 — envelope with a mandatory `identifier` (idempotencyKey).**

The payload **always** includes `identifier` — the event's idempotency key (aggregate id or unique event id). Consumers use `identifier` + `eventType` for automatic dedup. **[EVT-7]**

**EVT-8 — reactor in a different component from the emitter.**

A use-case never listens to its own event. The reactor is always another component (another module, another handler). **[EVT-8]**

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Type | Detector (🛠️) |
|---|---|---|---|
| EVT-1 | Name `<domain>.<action>` in the past; announces an accomplished fact | constitutional | Scenarios 1–2 / ❌ |
| EVT-3 | The publisher is the use-case (fire-and-forget, never await); every mutation emits; reads do not emit | constitutional | (see 04-use-cases ARC-CON-1) |
| EVT-4 | Side effect is a reaction via event; the emitter does not inject the side-effect service | constitutional | Scenario 3 / ❌ |
| EVT-5 | Reactor named after the intent (`sendXxxEmail`), never after the event (`onXxxCreated`) | constitutional | Scenario 3 / ❌ |
| EVT-6 | Payload: full entity in the simple case (zero-IO handler); normalized (ids only) at high volume | constitutional | Scenarios 1–2 / ❌ |
| EVT-7 | Payload always carries `identifier` (idempotencyKey); consumers use `identifier` + `eventType` for dedup | constitutional | Scenarios 1–2 |
| EVT-8 | Reactor in a different component from the emitter; a use-case never listens to its own event | constitutional | Scenario 3 |
| EVT-L1 | Reactor decorated with `@Subscriber('event')` and typed with `SubscriberPayload<'event'>` via `PublisherEventMap` augmentation | stack lint | Scenario 3 |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-CON-5` | Event only after the write confirms. |
| `ARC-CON-6` | Outbox when data and event must be atomic. |
| `ARC-IDM-2` | The dedup key comes from the fact. |
| `ARC-IDM-3` | The handler is commutative. |
| `ARC-IDM-5` | The event carries absolute state. |
| `ARC-CON-9` | A read replica is derived; the write declares what it invalidates. |

`ARC-CON-9` is what the event usually carries on this side: cache, read model,
search index and materialized counter are all derived from the write that has
just confirmed. The question "which reads went stale?" is answered together with
the write — by direct invalidation, TTL or event. What the law forbids is not
having answered it. And the symmetric holds: a business decision reads the
source, never the replica.


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into real code**: the standard is zero comments (see Code Quality).

**Registration (once, at the root)** — `@turystack/nestjs-publisher` is global; use-cases only inject `PublisherService`:

- **Standalone** → `PublisherModule.register({ adapter: 'event-emitter' })` — in-process delivery, same publishes, subscriber failure logged and isolated.
- **Monorepo** → `PublisherModule.register((config) => ({ adapter: 'aws', aws: { region, eventBridge: { busName, source } } }))` — `TOPIC` goes to EventBridge, `QUEUE` to SQS.

`publish` returns `void` and is **fire-and-forget**: delivery happens in the background, failure is logged by the lib — **never `await`**, never a try/catch around it. Serialization is superjson (Dates, Maps, Sets preserved). `flush()` belongs to the `Serverless.create` wrapper and to the lib's shutdown — **never** to app code.

**Typing** — the `PublisherEventMap` augmentation types `publish` and reactors at compile time:

```typescript
declare module '@turystack/nestjs-publisher' {
  interface PublisherEventMap {
    'invite.created': {
      destination: 'TOPIC'
      data: { identifier: string } & Invite
    }
  }
}
```

**Mechanisms per rule (NestJS):**

- **EVT-1** — string literal `'<domain>.<action>'` in the past as the publish `name`; the same literal is the key in `PublisherEventMap`.
- **ARC-CON-5** — `this.publisher.publish(...)` always after the operation's `await` (persist/computation); never before — and never with `await`.
- **EVT-3** — `PublisherService` injected only into use-cases; never into controllers, repositories or entities (see `04-use-cases`).
- **EVT-4** — a side-effect service (notification, audit) never appears in a use-case's `@Inject(...)`.
- **EVT-5** — the reactor's name expresses the action (`sendInviteEmail`, `notifyDriverAssigned`), not the event.
- **EVT-6** — simple case: spread the entity into `data`; high volume: only ids/keys in the object.
- **EVT-7** — `data: { identifier: entity.entityId, ...entity }` or `data: { identifier: entity.entityId, ...ids }`.
- **EVT-8** — reactor in a separate component, never alongside the use-case that published it.
- **EVT-L1** — `@Subscriber('invite.created') async sendInviteEmail(invite: SubscriberPayload<'invite.created'>)` — the decorator binds the event; the typing comes from `PublisherEventMap`. The optional `{ schema }` validates the payload with Zod before delivery (invalid → logged and discarded).

### ✅ How to do it

**Scenario 1 — simple case: publish the full entity:** `[EVT-1, ARC-CON-5, EVT-6, EVT-7]`
```typescript
const invite = await this.inviteRepository.save(input)
this.publisher.publish({
  destination: 'TOPIC',
  name: 'invite.created',
  data: { identifier: invite.inviteId, ...invite }, // identifier = idempotencyKey; no await
})
return invite
```

**Scenario 2 — high volume: normalized payload (ids only):** `[EVT-1, ARC-CON-5, EVT-6, EVT-7]`
```typescript
// many events per second → do not ship the whole entity
this.publisher.publish({
  destination: 'TOPIC',
  name: 'trip.materialized',
  data: { identifier: trip.tripId, scheduleId: trip.scheduleId },
})
```

**Scenario 3 — reactor in ANOTHER component, named after the intent:** `[EVT-4, EVT-5, EVT-8, EVT-L1]`
```typescript
@Injectable()
export class InviteNotificationsService {
  @Subscriber('invite.created')
  async sendInviteEmail(invite: SubscriberPayload<'invite.created'>) {
    try {
      this.logger.log('sendInviteEmail', { inviteId: invite.inviteId })
      await this.mailService.send({ to: invite.email, template: 'invite', data: invite })
    } catch (error) {
      this.logger.error('sendInviteEmail', error)
      throw error
    }
  }
}
```
> In the monorepo the reactor is an app handler `@Handler('EVENTBRIDGE')` — same intent in the name, same body (see `08-background-handlers`).

### ❌ Never do

```typescript
// ❌ [EVT-4] use-case injecting the notification service (it should publish an event)
constructor(@Inject(NotificationService) private readonly notification: NotificationService) {}

// ❌ [EVT-6] in the simple case, sending a thin payload and making the reactor fetch it again
this.publisher.publish({ destination: 'TOPIC', name: 'invite.created', data: { identifier: invite.inviteId } })
@Subscriber('invite.created')
async sendInviteEmail(payload) { const invite = await this.getInviteUseCase.execute(payload.identifier) }

// ❌ [EVT-5] named after the event, not after the intent
@Subscriber('invite.created') async onInviteCreated() {}

// ❌ [ARC-CON-5] publishing before success
this.publisher.publish({ destination: 'TOPIC', name: 'invite.created', data })
await this.inviteRepository.save(input)

// ❌ (fire-and-forget — see 00-overview) awaiting the publish; it returns void and the lib logs the failure
await this.publisher.publish({ destination: 'TOPIC', name: 'invite.created', data })
```
