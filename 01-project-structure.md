# Project Structure & Domain Boundaries

**Concept.** The structure reflects business capabilities and delivery points.
Folders exist when they hold real code; the CLI does not create empty zones.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code and the tree in
> TypeScript · NestJS · @turystack. The split, `XXX-n` versus `XXX-Ln`, and
> why an `ARC-…` law is cited and never restated: `turystack-backend-pattern`
> › *How a section is written*.

---

**Rules defined here:** `PRJ-1` · `PRJ-2` · `PRJ-3` · `PRJ-L1` — the law is
the *Invariants* table below; every ❌ item cites the id it violates.

## 🌐 Generic pattern (portable — stack-independent)

**PRJ-1 — the domain's internal dependency order.**

Inside a domain the order is contract → entity → persistence → operation →
delivery: schema, then entity, then repository, then use-case, then
controller/handler. It is `ARC-LAY-1` at the granularity of one domain folder,
and it is what makes the domain testable with no infrastructure. **[PRJ-1]**

**PRJ-2 — a shared capability is registered once, at the consuming app's root.**

A module the whole app depends on — database, logger, IAM, cache — is registered
at the root of the app that consumes it, never re-registered per domain and
never wrapped in a local module that only re-exports it. A second registration
means a second instance, a second connection pool and a configuration that
drifts. **[PRJ-2]**

**PRJ-3 — a shared artifact never lives inside a delivery app.**

Schema, migration, entity and use-case belong to the shared package. An app
folder holds its delivery point, its own configuration and nothing another app
would need. The moment a second app needs the artifact, an app-owned copy is a
fork. **[PRJ-3]**

### Invariants (the law the gates enforce)


| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| PRJ-1 | Inside a domain: schema → entity → repository → use-case → controller/handler | constitutional | `gate:folder-shape` | Domain anatomy / ❌ |
| PRJ-2 | A shared capability is registered once at the consuming app's root, never wrapped again | constitutional | `gate:single-registration` | Dependency injection / ❌ |
| PRJ-3 | Schema, migration, entity and use-case live in a package, never inside a delivery app | constitutional | `gate:shared-artifact-placement` | The tree / ❌ |
| PRJ-L1 | `@turystack/backend-config` is the source of truth for lint, format and TypeScript; a project never redefines those rules locally | stack lint | `gate:config-extends` | Dependency injection / ❌ |

Ids are stable across versions. A gap in the numbering is a law that moved to
the constitution — the table below says where it went.

## Governed by the constitution

These laws live in `turystack-architecture-pattern` and are not restated here.
What follows is how the Turystack backend expresses them.

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-LAY-2` | The domain imports no infrastructure. | `domains/` imports no controller, no Nest transport, no adapter |
| `ARC-LAY-4` | Cross-module access only through the public operation. | domain B calls domain A's use-case, never `A`'s repository |
| `ARC-LAY-5` | The barrel exposes the public surface. | `domains/{domain}/index.ts`; no import reaching inside another domain |
| `ARC-LAY-6` | Organization by domain/feature; no mandatory technical-layer folder. | `domains/{capability}/`, never `src/infrastructure/` |
| `ARC-LAY-7` | A helper lives with its owner. | `support/` only for ownerless pure functions |
| `ARC-LAY-8` | Infra covered by a lib is used directly, never wrapped. | inject the lib's service; no `CacheModule` of your own |
| `ARC-DEL-1` | The delivery boundary is thin. | `controllers/` translate and delegate |
| `ARC-TOP-3` | An app registers only the provider closure it consumes. | each `app.module.ts` registers its own closure |
| `ARC-TOP-4` | A folder exists when it has real code. | no `.gitkeep` layer placeholders |
| `ARC-TOP-6` | Config is validated at boot and read through a typed service. | `config.schema.ts` + injected config; no `process.env` downstream |

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack)

### The tree

There is one tree. A product with a single API and a product with six apps have
the same shape, because the domain is a package either way — the second app
costs a folder, never a refactor.

```text
apps/
├── api/                          delivery: HTTP, one audience per surface
│   └── src/
│       ├── controllers/
│       │   ├── auth/             sign-in, sign-up, social
│       │   └── order/
│       │       ├── order.controller.ts
│       │       └── order.schemas.ts
│       ├── auth/                 the IAM profile bridge
│       ├── app.module.ts
│       ├── config.schema.ts
│       └── main.ts
├── auth/                         the sign-in application
├── admin/                        one app per audience
└── process-payment/              delivery: a queue handler
    └── src/
        ├── process-payment.handler.ts
        ├── process-payment.module.ts
        ├── config.schema.ts
        └── main.ts
domains/                          one package per domain
├── identity/                     @acme/identity
└── order/                        @acme/order
    ├── package.json
    └── src/
        ├── order.schema.ts
        ├── order.types.ts
        ├── order.entity.ts
        ├── order.repository.ts
        ├── order.mock.ts
        ├── use-cases/
        │   └── create-order/
        │       ├── create-order.ts
        │       ├── create-order.types.ts
        │       └── create-order.test.ts
        └── index.ts
packages/                         shared: backend, frontend, or both
├── exceptions/                   @acme/exceptions — the one catalogue
├── database/                     @acme/database — schema, relations, drizzle/
├── ui/                           @acme/ui — the theme, as one stylesheet
├── oauth-clients/                @acme/oauth-clients — who may sign a person in
└── support/                      only once the reuse actually exists
```

- `adapters/` is born only when an integration not covered by the libs exists.
- `app.module.ts` registers controllers, the provider closure and global modules.
- `main.ts` contains only `Server.create`.
- A multi-audience API adds only the `controllers/{audience}/{domain}/` level.
  It stays one app: `ARC-TOP-1` gives an app its own lifecycle, and two
  audiences that deploy together do not have one.

### The scope

Every package the repository owns is scoped by the repository's own name —
`@acme/order`, `@acme/database` — and `acme` above stands in for whatever the
product is called. A fixed scope such as `@repo` reads the same in every
repository on the machine, which is exactly when it stops carrying information:
the import says a package is internal, and nothing about which product it is
internal to.

### One package per domain

A domain is a package, not a folder. That is what turns `ARC-LAY-4` from a rule
someone has to notice into something the toolchain refuses:

- Depending on another domain is a line in `package.json` —
  `"@acme/billing": "workspace:*"` — so the dependency is declared where
  dependencies are read, and it shows up in review as one.
- A cycle between two domains fails `tsc -b`. The build orders the packages by
  their references, and a reference cycle has no order.
- Reaching past a barrel is a resolution error, not a convention: `@acme/order`
  resolves to what `index.ts` exports and to nothing else.

The catalogue lives in `packages/exceptions` for the same reason. Kept inside
one domain, every other domain would have to depend on that domain just to raise
an error.

- `@acme/<domain>` holds that domain's schema, entity, repository, use-cases and
  events — and exports the use cases, which is its public surface.
- `@acme/exceptions` holds the product's one error catalogue.
- `@acme/database` holds the schema, relations, migrations and the
  `DatabaseService` augmentation.
- APIs and handlers are delivery apps: they compose modules and import
  operations from the domain packages they actually deliver.
- Each app has its own configuration and registers only the libs it needs.

### Domain anatomy

Each file has a role. An exclusive 1:1 sub-resource belongs to the aggregate; it
does not become a separate domain. A use-case lives in `use-cases/{operation}/`.
The barrel exposes only the domain's public API.

A local repository is created when it adds domain policy or composition: a
multi-table aggregate, a specific query, soft delete or entity hydration. For
trivial operations already covered by the `DatabaseService` typed repository, do
not add an interface and a wrapper with no behavior.

`support/` is allowed for pure functions with no domain owner — for example,
cross-cutting normalization used by several domains and still specific to the
product. It takes no business rule, infrastructure client, logger,
configuration or lib wrapper. Code shared across apps is born as an explicit
package (`packages/support`) only once the reuse exists — never in advance.

### Dependency injection

- Global lib → service/token provided by the lib itself.
- Swappable local adapter → interface + `Symbol`.
- Use-case → class.
- Controller/handler → use-case.
- Use-case → its own repository, other domains' use-cases and lib/adapter
  services.

Use package/alias imports; never a relative path crossing a domain. Lint, format
and TypeScript rules come from `@turystack/backend-config` — including import
ordering. A local `biome.json`/`tsconfig.json` extends it and never redefines
its rules (`PRJ-L1`).

### Which library owns which concern

The skill decides **where code belongs**; the library decides **how its API is
called**. Before writing infrastructure by hand, check whether it already has an
owner (`ARC-LAY-8`):

| Concern | Owner |
|---|---|
| HTTP/bootstrap, app composition | `@turystack/nestjs-server` |
| Serverless delivery point, scheduled or event-driven | `@turystack/nestjs-serverless` |
| Typed config validated at boot | `@turystack/nestjs-config` |
| Database, typed repositories, migrations | `@turystack/nestjs-database` |
| Filters, pagination, sorting over a query | `@turystack/query-dsl` |
| Input field schemas shared by body and form | `@turystack/fields` |
| Entity base and guards | `@turystack/entity` |
| Error catalogue and categories | `@turystack/exceptions` |
| Authentication, permissions, ACL | `@turystack/nestjs-iam` |
| Social/OAuth sign-in | `@turystack/nestjs-social-auth` |
| Request-scoped context and correlation | `@turystack/nestjs-context` |
| Structured logging | `@turystack/nestjs-logger` |
| Metrics, traces, error reporting | `@turystack/nestjs-observability` |
| Publishing and consuming events | `@turystack/nestjs-publisher` |
| Cache / read replica | `@turystack/nestjs-cache` |
| Distributed lock, mutual exclusion (`ARC-DEL-7`) | `@turystack/nestjs-lock` |
| Idempotency key and replay protection (`ARC-IDM-6`) | `@turystack/nestjs-idempotency` — decisions in `15-idempotency.md` |
| Timeout, retry, circuit breaker (`ARC-RES-*`) | `@turystack/nestjs-resilience` — decisions in `14-resilience.md` |
| Rate limiting | `@turystack/nestjs-rate-limit` |
| File storage | `@turystack/nestjs-storage` |
| Compensation across steps (`ARC-CON-1`) | `@turystack/saga` |
| Lint, format, TypeScript | `@turystack/backend-config` |

An integration with no owner in this table sits behind an interface owned by the
application (`ARC-LAY-8`) — see `07-adapters.md`.

### ❌ Never do

- `[ARC-TOP-4]` Create empty folders to represent layers.
- `[ARC-LAY-8]` Create `src/infrastructure/cache` or any adapter wrapping a Turystack lib.
- `[ARC-LAY-7]` Use `src/support` as a generic destination for ownerless code.
- `[PRJ-2]` Create `OrderModule` just to re-export the domain's providers, or register a global lib a second time inside a domain.
- `[PRJ-3]` Put a schema, a migration, an entity or a use case inside a delivery app.
- `[ARC-TOP-6]` Read `process.env` in a controller, use-case, repository, adapter or `main.ts`.
- `[PRJ-1]` Have an entity import its repository, or a repository import a use-case.
- `[PRJ-L1]` Redefine lint/format/TypeScript rules locally instead of extending `@turystack/backend-config`.
