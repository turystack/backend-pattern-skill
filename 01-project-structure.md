# Project Structure & Domain Boundaries

**Concept.** The structure reflects business capabilities and delivery points.
Folders exist when they hold real code; the CLI does not create empty zones.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code and the tree in
> TypeScript · NestJS · @turystack. The split, `XXX-n` versus `XXX-Ln`, and
> why an `ARC-…` law is cited and never restated: `turystack-backend-pattern`
> › *How a section is written*.

---

**Rules defined here:** `PRJ-1` · `PRJ-2` · `PRJ-3` · `PRJ-4` · `PRJ-5` ·
`PRJ-L1` — the law is
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

**PRJ-4 — every aggregate owns its own contract, and no file covers them all.**

A domain with one aggregate keeps its files at its root. A domain with several
gives each its own folder, and each folder declares the row it owns and the
closed sets that row uses. There is no schema or types file for the whole
domain: a file every folder imports from is a file every folder is coupled to,
and the enum an operation needs stops being findable from the operation that
needs it. A row is inferred where it is used rather than published beside the
entity under a second name — the entity already holds that name, and two names
for one shape is the pair that drifts. **[PRJ-4]**

**PRJ-5 — an operation is a folder, and the shape it accepts lives in it.**

One operation, one folder: the operation, the shape it accepts and a barrel. The
barrel is what the domain's own index imports, so adding a file to an operation
never changes the line that exports it. The shape belongs to the operation
rather than to the aggregate, because it is what the boundary accepts and not
what the table holds. **[PRJ-5]**

### Invariants (the law the gates enforce)


| ID | Law (one line) | Class | Gate | Detector (🛠️) |
|---|---|---|---|---|
| PRJ-1 | Inside a domain: schema → entity → repository → use-case → controller/handler | constitutional | `gate:folder-shape` | Domain anatomy / ❌ |
| PRJ-2 | A shared capability is registered once at the consuming app's root, never wrapped again | constitutional | `gate:single-registration` | Dependency injection / ❌ |
| PRJ-3 | Schema, migration, entity and use-case live in a package, never inside a delivery app | constitutional | `gate:shared-artifact-placement` | The tree / ❌ |
| PRJ-4 | A domain with more than one aggregate gives each its own folder under `entities/`; no schema or types file covers the whole domain | constitutional | `gate:domain-anatomy` | Domain anatomy / ❌ |
| PRJ-5 | An operation is a folder holding the operation and a barrel; its input contract lives there too | constitutional | `gate:domain-anatomy` | Domain anatomy / ❌ |
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
        ├── support/
        │   └── order.exceptions.ts
        ├── use-cases/
        │   └── create-order/
        │       ├── create-order.ts
        │       ├── create-order.types.ts
        │       └── create-order.test.ts
        └── index.ts
libs/                             shared: backend, frontend, or both
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

Each domain publishes its own codes, under its own name (`ARC-ERR-1`): the
prefix is the domain's, one package cannot be two domains, and the code is
declared beside the rule that raises it.

- `@acme/<domain>` holds that domain's schemas, entities, repositories, use
  cases and codes — and exports the use cases, the schemas and the codes, which
  is its public surface.
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

### A domain with more than one aggregate

The tree above is a domain with one aggregate, and most have one. A domain that
holds several — identity holds the person, the organizations they act for, the
roles those carry and the codes they sign in with — gives each its own folder
under `entities/`, with the files the single-aggregate domain keeps at its root:

```text
domains/iam/src/
├── entities/
│   ├── user/
│   │   ├── user.schema.ts        the row it owns, and the closed sets that row uses
│   │   ├── user.types.ts         what those sets are called in TypeScript
│   │   ├── user.entity.ts        the invariants, and the helpers that serve them
│   │   ├── user.repository.ts    rows in and out
│   │   ├── user.mock.ts          the builder a test writes with
│   │   └── index.ts              the way in, from any other folder
│   ├── organization/ · membership/ · otp/ · role/
│   └── permission/ · workspace/  the same six files, with no exception
├── support/
│   ├── iam.permissions.ts        what no single aggregate owns
│   ├── iam.exceptions.ts         the codes this domain publishes
│   └── iam.seed.ts               brings the database in line with the catalogue
├── use-cases/
│   └── sign-up/
│       ├── sign-up.schema.ts     the shape the operation accepts
│       ├── sign-up.types.ts      its input, inferred from that schema
│       ├── sign-up.ts            the operation
│       ├── sign-up.test.ts
│       └── index.ts              what the domain's own barrel imports
└── index.ts
```

- **No file covers the whole domain.** An `iam.schema.ts` holding every row is a
  file every folder imports from, and the enum an operation needs stops being
  findable from the operation (`PRJ-4`).
- **A row is not published beside its entity.** `User` is the entity; a
  `UserRecord` next to it names the same shape twice. Where the row is needed —
  the entity's constructor, the repository's cast, the mock's overrides — it is
  `z.infer<typeof userSchema>`, derived in the file that needs it.
- **Every aggregate is the same six files.** Schema, types, entity, repository,
  mock, barrel — including the ones that look like they need less. A workspace
  with no repository is a use case reaching the table directly, which is what
  the layering forbids; an aggregate with no entity is a row with nowhere to put
  its invariants.
- **A pure helper lives in its entity, as a static.** Hashing a password is
  `User.hash`, slugging a name is `Organization.slugify`, minting a code is
  `Otp.generateCode`. A `user.password.ts` beside the entity is a second place
  the reader has to find, and the entity is already the thing that owns the
  rule. What no aggregate owns — the scrypt primitive both the password and the
  code use — goes to `support/`.
- **A bootstrap is not an operation.** The seed exists to make the database
  agree with the catalogue in the source; nobody calls it to get something done,
  and it belongs beside the catalogue it mirrors rather than in `use-cases/`.
- **The operation's contract is the operation's.** `sign-up.schema.ts` sits with
  `sign-up.ts`; the row schema sits with the aggregate whose row it describes
  (`PRJ-5`).
- **What leaves the package is the contracts and the operations.** The entities
  and the repositories are how the domain works, not what it offers — the barrel
  exports neither (`ARC-LAY-4`, `ARC-LAY-5`).
- **The order still runs one way** (`PRJ-1`): an aggregate's schema and types are
  imported by its entity, its repository and any operation; an operation's schema
  is imported by that operation and by the contracts barrel, and by nothing else.

Two operations here have no `*.types.ts`, and that is the rule rather than an
exception to it: `resolve-profile` implements a signature the IAM library owns
and `seed-iam` takes nothing. A file that exists to satisfy a pattern rather
than to hold something is the `.gitkeep` `ARC-TOP-4` refuses.

`support/` is allowed for pure functions with no domain owner — for example,
cross-cutting normalization used by several domains and still specific to the
product. It takes no business rule, infrastructure client, logger,
configuration or lib wrapper. Inside a domain the same word means the same
thing one level down: what no single aggregate owns, such as a catalogue that
spans three of them. What one aggregate could own belongs to that aggregate. Code shared across apps is born as an explicit
package (`libs/support`) only once the reuse exists — never in advance.

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
- `[PRJ-4]` Keep one `<domain>.schema.ts` or `<domain>.types.ts` for a domain that holds several aggregates.
- `[PRJ-4]` Publish a `UserRecord` beside the `User` entity — infer the row where it is used.
- `[PRJ-5]` Put an operation's input schema in the aggregate's schema file, or leave an operation without its barrel.
- `[PRJ-L1]` Redefine lint/format/TypeScript rules locally instead of extending `@turystack/backend-config`.
