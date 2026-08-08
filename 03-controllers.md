# Controllers

**Concept.** Transport boundary: it translates HTTP ↔ application call. Thin — it validates the input, delegates to the use-case, returns the response. No business rule, no data access, no transaction.

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `CTL-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `CTL-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**ARC-6 — thin controller.**

The controller validates the input shape at the edge, delegates execution to the use-case (typically 1 route → 1 use-case) and returns the response. It never contains a business rule, never accesses a repository or adapter directly, never runs a transaction. **[ARC-6]**

**CTL-2 — multiple surfaces.**

A codebase may expose several **HTTP surfaces**, one per consumer (e.g. customer app, operations console, partner app, backoffice), each with its own route prefix and its own OpenAPI document. Webhooks from external providers live in an isolated surface, separate from the consumption APIs. The logic lives in the shared domain layer; the controller is only the per-surface edge. **[CTL-2]**

**ARC-SEC-1 — scope from the auth context, never from client input.**

On every surface with an authenticated user, the scope identifier (`organizationId`, `userId`) is extracted from the authenticated context and injected by the controller — **never** trusted from the client's body/query/params:

- **Client surface** → scope **forced** from the authenticated context (the identity itself), never from input.
- **Operator surface** → the **operator organization's** scope, forced by the permission; limited to their org.
- **Internal surface** → **no** forced scope; the organization identifier (and the like) comes in as an **optional query** in the standard input, since the internal user queries **across** organizations.

The use-case receives the id and asserts/filters internally. **[ARC-SEC-1]**

**CTL-4 — permission gating (mandatory on every protected route).**

Every route that requires authenticated access must combine two levels of protection:

1. **Surface access** — checks whether the profile has permission to access the API (e.g. `access:bko`, `access:ops`). Applies to **all** endpoints of the surface.
2. **Resource permission** — checks whether the profile has permission for the specific resource (e.g. `invite:read`) **and** restricts the scope (see ARC-SEC-1). Applies per route.

Fully protected controllers stack both; controllers with a public/protected mix omit the surface level from the class and apply it per route. **[CTL-4]**

**CTL-5 — endpoint ordering by HTTP method.**

Inside the controller, endpoints are always listed in this order: GET → POST → PUT → PATCH → DELETE. **[CTL-5]**

**CTL-6 — terse route documentation.**

Each route's `summary`/`description` are terse: action + noun, with no scope qualifiers ("of the organization", "for operators", "in the global catalogue"). **[CTL-6]**

**CTL-7 — business action = custom method with `:verb` (ALWAYS, MANDATORY).**

A route that performs an **action** on a resource (not plain CRUD) uses the custom method pattern: the verb goes into the path **separated by `:`**, attached to the resource — `POST /invoices/:invoiceId:pay`, `POST /orders/:orderId:cancel`, `POST /users/:userId:activate`. Never the verb as a segment (`/invoices/:invoiceId/pay`): a path segment is a **resource/relationship** (`/workspaces/:workspaceId/invites`), never an action. The custom method's HTTP method is always `POST`. **[CTL-7]**

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Type | Detector (🛠️) |
|---|---|---|---|
| CTL-2 | Multiple surfaces with their own prefix/doc; webhooks in an isolated surface | constitutional | Scenarios 1–4 |
| CTL-4 | Every protected route stacks surface-access + resource-permission with scope | constitutional | Scenarios 1–2, 4 / ❌ |
| CTL-5 | Endpoints ordered: GET → POST → PUT → PATCH → DELETE | constitutional | Scenario 3 / (see 00-overview) |
| CTL-6 | Terse summary/description: action + noun; no scope qualifiers | constitutional | Scenarios 1–4 / ❌ |
| CTL-7 | Business action = custom method `:verb` in the path (`/invoices/:invoiceId:pay`), always POST; the verb never becomes a segment | constitutional | Mechanism / ❌ |
| CTL-L1 | Main schemas registered via `schemas:{}` on the controller decorator (reusable model in the docs) | stack lint | Scenario 1 |
| CTL-L2 | DTOs derive from canonical schemas via `.pick`/`.extend`; only `*Request`/`*SchemaResponse` exported | stack lint | Scenario 4 |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-6` | The delivery boundary is thin. |
| `ARC-SEC-1` | Scope comes from the authenticated context. |
| `ARC-SEC-3` | Authorization is operation plus resource. |


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · @turystack):**

- **ARC-6** — `@Inject(CreateOrderUseCase)`; the route handler calls `useCase.execute(input)` (typically 1 route → 1 use-case); the controller never injects a repository, `DatabaseService` or direct adapters. There is never a business-rule `if`/`throw` nor `@Transactional` in the controller.
- **CTL-2** — `@Controller({ path, prefix, tag, schemas? })` from `@turystack/nestjs-server`; `prefix` defines the **project** (`app`, `admin`, `internal`). Each project goes into the `projects` array of `Server.create` and gets its own OpenAPI document + Scalar reference (`/{globalPrefix}/v1/{prefix}/openapi` · `/reference`). Single-project API: no `prefix`, controllers in `src/controllers/{domain}/`; multi-project: `src/controllers/{project}/{domain}/`.
- **ARC-SEC-1** — `@AuthenticatedProfile() profile: IamProfile` (from `@turystack/nestjs-iam`) extracts the authenticated scope; the client surface injects `profile.userId`/`profile.organizationId` explicitly into the use-case input; a route scoped to a workspace declares the context in `@ACL('resource:action', (request) => ({ workspaceId: request.params.workspaceId }))` — IAM cross-checks the declared context against the conditions of the profile's role; the internal surface receives `organizationId` as an optional field in the query schema.
- **CTL-4** — `@Auth()` at the class level (fully protected controller) **and** `@ACL('resource:action', contextCallback?)` per route — `@ACL` already stacks authentication + permission check. In a controller with a public/protected mix, omit `@Auth` from the class and apply it per route.
- **CTL-5** — ordering convention (see `00-overview`).
- **CTL-6** — `@Route({ summary: 'Create Order', description: 'Creates an order.' })`; never "Creates an order in the organization catalogue."
- **CTL-7** — turystack syntax in `@Route`: the verb comes in with **`::`** — `@Route({ method: 'POST', path: ':invoiceId::pay', ... })` matches `POST /invoices/inv-123:pay` and extracts `invoiceId`; collection custom methods work the same (`'reports::generate'`). The decorator internally converts it to the escape the Express 5 path matcher requires (`\:` — the documented NestJS 11 pattern for reserved characters) and **validates at boot**: `::` with a method other than POST throws `[Route] custom method path '...' requires method POST`. No escape ever appears in project code.
- **CTL-L1** — `@Controller({ schemas: { Order: { schema: orderResponse }, OrderStatus: { schema: orderStatusSchema }, OrderIdentity: { schema: orderIdentitySchema } } })` — registers the domain's main schemas as named models in the OpenAPI document; never anonymous inline.
- **CTL-L2** — `createRequestSchema({ body: orderSchema.pick({...}) })` / `.extend({...})` in the `{domain}.schemas.ts` next to the controller; exports only `*Request` / `*SchemaResponse` (the name must contain `Schema`); bodies inlined and derived from the domain schemas.

### ✅ How to do it

**Scenario 1 — protected route, injects the use-case, terse summary, schemas in OpenAPI:** `[ARC-6, CTL-4, CTL-6, CTL-L1]`
```typescript
@Controller({
  path: 'invoices',
  prefix: 'admin',
  tag: 'Invoice',
  schemas: {
    Invoice: { schema: invoiceResponse },
    InvoiceStatus: { schema: invoiceStatusSchema },
    InvoiceIdentity: { schema: invoiceIdentitySchema },
  },
})
@Auth()
export class InvoiceAdminController {
  constructor(@Inject(CreateInvoiceUseCase) private readonly createInvoiceUseCase: CreateInvoiceUseCase) {}

  @Route({ method: 'POST', path: '', summary: 'Create Invoice', description: 'Creates an invoice.' })
  async create(@Request(createInvoiceRequest) req: RequestInput<typeof createInvoiceRequest>) {
    return this.createInvoiceUseCase.execute(req.body)
  }
}
```

**Scenario 2 — resource authorization with the scope declared in the ACL:** `[ARC-SEC-1, CTL-4]`
```typescript
@Route({ method: 'GET', path: 'workspaces/:workspaceId/invites', summary: 'List Invites', description: 'Lists invites.' })
@ACL('invite:read', (request) => ({ workspaceId: request.params.workspaceId }))
async list(
  @AuthenticatedProfile() profile: IamProfile,
  @Request(listInvitesRequest) req: RequestInput<typeof listInvitesRequest>,
) {
  return this.listInvitesUseCase.execute({ ...req.query, organizationId: profile.organizationId })
}
```

**Scenario 3 — client API: scope injected from `@AuthenticatedProfile`, never from the input:** `[ARC-SEC-1, CTL-4, CTL-5]`
```typescript
@Controller({ path: 'orders', prefix: 'app', tag: 'Order' })
@Auth()
export class OrderAppController {
  constructor(
    @Inject(ListOrdersUseCase) private readonly listOrdersUseCase: ListOrdersUseCase,
    @Inject(CreateOrderUseCase) private readonly createOrderUseCase: CreateOrderUseCase,
  ) {}

  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  async list(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
  ) {
    return this.listOrdersUseCase.execute({ ...req.query, userId: profile.userId })
  }

  @Route({ method: 'POST', path: '', summary: 'Create Order', description: 'Creates an order.' })
  async create(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(createOrderRequest) req: RequestInput<typeof createOrderRequest>,
  ) {
    return this.createOrderUseCase.execute({
      ...req.body,
      organizationId: profile.organizationId,
    })
  }
}
```

**Scenario 4 — same resource, three projects (app / admin / internal):** `[CTL-2, ARC-SEC-1, CTL-4, CTL-L2]`
```typescript
// main.ts — each project in the projects array gets its own prefix + OpenAPI doc + Scalar reference
Server.create(AppModule, (config) => ({
  port: config.get('PORT'),
  title: 'Acme API',
  description: 'Acme backend',
  docs: { provider: 'scalar' },
  projects: [
    { name: 'app', title: 'App API', prefix: 'app' },
    { name: 'admin', title: 'Admin API', prefix: 'admin' },
    { name: 'internal', title: 'Internal API', prefix: 'internal' },
  ],
}))
```

```typescript
// App (client) — scope FORCED from @AuthenticatedProfile
@Controller({ path: 'orders', prefix: 'app', tag: 'Order' })
@Auth()
export class OrderAppController {
  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  async list(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
  ) {
    return this.listOrdersUseCase.execute({ ...req.query, userId: profile.userId })
  }
}

// Admin (operator) — resource permission via @ACL; the operator organization's scope
@Controller({ path: 'orders', prefix: 'admin', tag: 'Order' })
@Auth()
export class OrderAdminController {
  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  @ACL('order:read')
  async list(
    @AuthenticatedProfile() profile: IamProfile,
    @Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>,
  ) {
    return this.listOrdersUseCase.execute({
      ...req.query,
      organizationId: profile.organizationId,
    })
  }
}

// Internal — no forced scope; organizationId is an OPTIONAL query in the standard input
@Controller({ path: 'orders', prefix: 'internal', tag: 'Order' })
@Auth()
export class OrderInternalController {
  @Route({ method: 'GET', path: '', summary: 'List Orders', description: 'Lists orders.' })
  async list(
    @Request(listOrdersInternalRequest) req: RequestInput<typeof listOrdersInternalRequest>,
  ) {
    return this.listOrdersUseCase.execute(req.query)
  }
}
```

```typescript
// internal: organizationId comes in as an OPTIONAL query, in the {domain}.schemas.ts next to the controller
export const listOrdersInternalRequest = createRequestSchema({
  query: z.object({
    organizationId: orderSchema.shape.organizationId.optional(),
    status: orderStatusSchema.optional(),
    page: z.coerce.number().int().min(1).default(1),
    pageSize: z.coerce.number().int().min(1).max(50).default(20),
  }),
})
```

## Transport contracts

- The domain's main schema is canonical. Requests derive from it with
  `pick`/`omit`/`extend`, without redeclaring fields.
- The request schema sits next to the controller because it belongs to the HTTP boundary.
- Export only contracts that another boundary reuses.
- The response represents an entity or an explicit public schema; it never leaks
  a database row, a secret or an internal field.
- Route params, query and body are validated before the use-case.
- An incompatible contract change requires a new version or a transition strategy;
  silently renaming a field is not allowed.

The concrete request/response helpers and the OpenAPI generation belong to the
`@turystack/nestjs-server` documentation.

### ❌ Never do

```typescript
// ❌ [ARC-6] injecting a repository/DatabaseService/adapter into the controller
constructor(@Inject(OrderRepository) private readonly orderRepository: OrderRepository) {}

// ❌ [ARC-6] business rule / transaction in the controller
async create(@Request(s) req) {
  if (await this.orderRepository.existsByCode(req.body.code)) throw new ConflictException(...)
}

// ❌ [CTL-6] summary with a scope qualifier
description: 'Lists invites of the organization for operators.'  // use 'Lists invites.'

// ❌ [CTL-7] action as a path segment — a segment is a resource, never a verb
@Route({ method: 'POST', path: ':invoiceId/pay', ... })      // use ':invoiceId::pay'

// ❌ [CTL-7] custom method outside POST — the decorator throws at boot
@Route({ method: 'GET', path: ':invoiceId::pay', ... })

// ❌ [CTL-7] business action disguised as CRUD instead of a custom method
@Route({ method: 'PATCH', path: ':orderId', ... })           // body { status: 'canceled' } → use ':orderId::cancel'

// ❌ [ARC-SEC-1] in a client API, trusting the scope coming from the client (forgeable)
async list(@Request(listOrdersRequest) req: RequestInput<typeof listOrdersRequest>) {
  return this.listOrdersUseCase.execute({ userId: req.query.userId }) // userId came from the client!
}

// ❌ [ARC-SEC-1] accepting organizationId/userId in the DTO of a client API
body: orderSchema.pick({ items: true, organizationId: true })
```
