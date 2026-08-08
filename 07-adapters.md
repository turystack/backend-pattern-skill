# Adapters

**Concept.** Ports & adapters: every external dependency (storage, email, payment gateway, OAuth, maps) sits behind a **main interface** owned by the application. An **adapter implements a concrete external provider** obeying that interface; the core depends on the abstraction, the provider is a swappable detail.

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Drizzle · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `ADP-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `ADP-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**ARC-14 — interface owned by the application.**

Every external dependency (storage, email, payment gateway, OAuth) sits behind an **interface owned by the application** — not by the provider. The core depends on the abstraction; the provider is a detail swappable by configuration or injection. **[ARC-14]**

**ADP-2 — injection via token/interface; never direct instantiation.**

The adapter is never instantiated by the caller. It is **injected by DI via token/interface**. The caller declares the dependency by the contract — not by the concrete implementation. **[ADP-2]**

**ADP-3 — no business logic; no knowledge of the domain schema.**

The adapter contains **no** business rule and does not import a domain schema. Its only role is to adapt the external provider's input/output to the interface contract. **[ADP-3]**

**ADP-4 — provider client received via DI; never created inside the concrete implementation.**

The external provider's client (e.g.: S3Client, SMTPTransport) is created externally and **injected** into the concrete implementation. The concrete implementation **never** instantiates the client inside itself. **[ADP-4]**

**ADP-5 — strategy with multiple providers: selection via a map; never an inline if/switch.**

When the adapter supports multiple providers, selection goes through **`Record<Provider, IAdapter>`**; the service indexes directly by the key. A provider if/switch scattered across the code is forbidden. **[ADP-5]**

**ADP-6 — three forms of adapter; always question which one fits.**

1. **Direct adapter (1 provider)** — fixed provider; the service just delegates.
2. **Strategy via payload `(provider, input)`** — the caller passes the `provider`; the service selects by that key.
3. **Strategy via internal business rule (no provider in the payload)** — the service resolves the provider **internally** (config/SystemSettings/rule); the caller does **not** pass the provider.

Strategies are a form of adapter and defer the resolution details here (see `07-adapters.md`). **[ADP-6]**

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Type | Detector (🛠️) |
|---|---|---|---|
| ADP-2 | Adapter injected via token/interface; never instantiated directly by the caller | constitutional | Forms 1–3 / ❌ |
| ADP-3 | Adapter with no business logic; does not know the domain schema | constitutional | ❌ |
| ADP-4 | Provider client received via DI; never created inside the concrete implementation | constitutional | Form 1 / ❌ |
| ADP-5 | Strategy via `Record<Provider, IAdapter>`; never an inline if/switch | constitutional | Forms 2–3 / ❌ |
| ADP-6 | Three forms: direct / strategy-payload / strategy-internal; question which one; Strategies = a form of adapter (see 07-adapters.md) | constitutional | Forms 1–3 |
| ADP-L1 | Structure: `src/adapters/{name}/` with `.interface.ts`, `.types.ts`, `.service.ts`, `.module.ts` (optional), `{provider}/{provider}.adapter.ts` | stack lint | Form 1 |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-14` | Lib used directly; integration without a lib behind an interface of its own. |
| `ARC-LAY-2` | Domain declares what it needs, not who does it. |


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into real code**: the standard is zero comments (see Code Quality).

**Library × adapter boundary (decide BEFORE creating any adapter).** Infra that a `@turystack` lib covers — cache (`nestjs-cache`), lock (`nestjs-lock`), rate-limit (`nestjs-rate-limit`), storage (`nestjs-storage`), tokens/IAM (`nestjs-iam`), social-auth (`nestjs-social-auth`), messaging (`nestjs-publisher`), database (`nestjs-database`), logs (`nestjs-logger`) — **never becomes an adapter**: register the lib's module **once at the root** and inject its service. The lib **already is** the interface owned by the application (ARC-14), with the provider as a detail swappable by configuration. `src/adapters/` exists **only** for an integration no lib covers — payment gateway, maps, postal-code lookup — and those follow the pattern below in full.

**Mechanisms per rule (NestJS):**

- **ARC-14** — `export const PAYMENT_ADAPTER = Symbol('PAYMENT_ADAPTER')` + `export interface IPaymentAdapter`; the service depends on `IPaymentAdapter`, never on the concrete implementation. Infra covered by a `@turystack` lib: the interface is already the lib's — do not create another one on top.
- **ADP-2** — `@Inject(PAYMENT_ADAPTER) private readonly adapter: IPaymentAdapter`; never `new StripeAdapter()`.
- **ADP-3** — the concrete implementation imports nothing from domains; the interface uses only types from `{name}.types.ts`.
- **ADP-4** — client bootstrap in `{name}.module.ts` via `useFactory`; the concrete implementation receives the ready client via `@Inject`.
- **ADP-5** — `@Inject(PAYMENT_ADAPTERS) private readonly adapters: Record<PaymentProvider, IPaymentAdapter>`; access via `this.adapters[provider]`.
- **ADP-6** — first ask: does a `@turystack` lib cover it? → then it is not an adapter. Then: fixed provider → Form 1; caller decides → Form 2; internal rule/config → Form 3.
- **ADP-L1** — directory structure checked in review; file names standardized by project convention.

```
src/adapters/{name}/
  {name}.adapter.interface.ts        # main interface + Symbol
  {name}.types.ts                    # input/output
  {name}.service.ts                  # delegates to the provider(s)
  {name}.module.ts                   # (optional) client bootstrap
  {provider}/{provider}.adapter.ts   # provider implementation (e.g.: stripe)
```

### ✅ How to do it

**Form 1 — direct adapter (1 provider):** `[ARC-14, ADP-2, ADP-6, ADP-L1]`
```typescript
@Injectable()
export class ZipCodeService {
  constructor(@Inject(ZIP_CODE_ADAPTER) private readonly adapter: IZipCodeAdapter) {}

  lookup(input: LookupZipCodeInput) {
    return this.adapter.lookup(input)
  }
}
```

**Form 2 — strategy via payload `(provider, input)`:** `[ARC-14, ADP-2, ADP-5, ADP-6]`
```typescript
@Injectable()
export class PaymentService {
  constructor(
    @Inject(PAYMENT_ADAPTERS)
    private readonly adapters: Record<PaymentProvider, IPaymentAdapter>,
  ) {}

  charge(provider: PaymentProvider, input: ChargeInput) {
    return this.adapters[provider].charge(input) // provider comes from the payload
  }
}
```

**Form 3 — strategy via internal business rule (no provider in the payload):** `[ARC-14, ADP-2, ADP-5, ADP-6]`
```typescript
@Injectable()
export class PaymentService {
  constructor(
    @Inject(PAYMENT_ADAPTERS)
    private readonly adapters: Record<PaymentProvider, IPaymentAdapter>,
    @Inject(SystemSettingsService)
    private readonly systemSettingsService: SystemSettingsService,
  ) {}

  async charge(input: ChargeInput) {
    const provider = await this.systemSettingsService.getPaymentProvider() // resolved internally
    return this.adapters[provider].charge(input)
  }
}
```

> Bootstrap (creating the client) lives in an optional `module` via `useFactory`; the concrete implementation receives the ready client via DI. **[ADP-4]**

### ❌ Never do

```typescript
// ❌ [ADP-2] service using the concrete implementation directly (must go through the interface/Symbol)
constructor(private readonly stripe = new StripeAdapter()) {}

// ❌ [ADP-3] adapter knowing the domain schema
import { productSchema } from '@/domains/product/product.schema'

// ❌ [ADP-5] provider if/switch scattered around (use Record<Provider, IAdapter>)
if (provider === 'STRIPE') { /* ... */ } else if (provider === 'PAGARME') { /* ... */ }

// ❌ [ADP-4] creating the client INSIDE the concrete implementation (it must receive it via DI)
class StripeAdapter { private client = new Stripe(process.env.STRIPE_SECRET_KEY) }

// ❌ [ADP-6] wrapping a @turystack lib in your own adapter — register the module at the root and inject the service
export const CACHE_ADAPTER = Symbol('CACHE_ADAPTER')
export class LibCacheAdapter implements ICacheAdapter {
  constructor(@Inject(CacheService) private readonly cache: CacheService) {}
}
```

🔵 **Proposed** — Retry / Timeout / Circuit Breaker in the adapter's service/module (timeout per call; retry with backoff; breaker per provider). ✍️ limits and libs to be provided.
