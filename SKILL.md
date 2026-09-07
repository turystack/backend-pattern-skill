---
name: turystack-backend-pattern
description: "How a Turystack backend is written — project structure and domain boundaries, Zod schemas, NestJS controllers, use-cases, repositories and persistence policy, entities, adapters, queue and scheduler handlers, domain events, the error catalogue, security and ACL, resilience, idempotency, tests and telemetry. Use it whenever you create, change or review backend code in a Turystack project: an endpoint or controller, a use-case, an entity guard, a repository, a migration, a consumer or cron job, a third-party integration, an error code, a permission or a test — including when the request sounds routine, like 'add a field', 'expose this data' or 'fix this validation', because those are the ones that quietly break a law. Read turystack-architecture-pattern first: it owns the law, this skill owns the mechanism and cites the law by ARC- id. For frontend code use turystack-frontend-pattern; the @turystack/* libraries own their setup and API docs."
---

# turystack-backend-pattern

Use this skill when creating, changing or reviewing backend application code.

## Prerequisite

`turystack-architecture-pattern` is the constitution and comes first. The law
lives there; this skill is how the backend expresses it. A constitutional law is
**never restated here** — it is cited by its id, which always starts with
`ARC-` (`ARC-LAY-4`, `ARC-CON-1`, `ARC-SEC-1`, `ARC-TST-3`…). When the two
disagree, the constitution wins.

Local ids (`PRJ`, `SCH`, `CTL`, `UC`, `REP`, `ENT`, `ADP`, `BGH`, `EVT`, `ERR`,
`TST`, `TEL`, `RSL`, `IDP`) are scoped to this skill. The frontend skill has its own `ERR`,
`TST` and `TEL`, and the two never cross.

## How to use

1. Read *Where things live* in `00-overview.md`. There is one shape — a domain
   is a package, a delivery is an app — so nothing is detected, only located.
2. Read `01-project-structure.md`. It also carries the *Which library owns which
   concern* table: check it **before** writing any infrastructure, because most
   of what looks like missing plumbing already has an owner (`ARC-LAY-8`).
3. Read the sections your task touches, from the routing table below — the whole
   section, not a remembered summary of it.
4. Consult the installed `@turystack/*` library documentation for setup, API,
   decorators and adapter options. Never reconstruct library usage from this
   skill.

## How a section is written

Every section splits in two, and the split is the point:

- **🌐 Generic pattern** — the portable law. Each rule has a stable id and, when
  the reason is not obvious, a paragraph explaining *why* — a rule you
  understand survives a refactor. It closes with the **Invariants** table, which
  is what a review binds to, and a **Governed by the constitution** table
  mapping each `ARC-…` law to its backend expression.
- **🛠️ Project-specific** — the same rules as TypeScript · NestJS · @turystack ·
  Drizzle · Zod code: mechanisms per rule, ✅ scenarios, and a **❌ Never do**
  block where every item carries the id it violates.

`XXX-n` is **constitutional** (it would survive a stack swap); `XXX-Ln` is a
**stack lint** (it exists because of this toolchain and would invert elsewhere —
still enforced here). Comments inside the examples are didactic: they explain
the rule, and never belong in real code.

The **Invariants** table is read column by column: the **id** a review binds
to, the **law** in one line, its **class** (`constitutional` or `stack lint`),
the **gate** that checks it, and the **detector** — how that gate catches the
violation, which is stack-specific and therefore lives in the 🛠️ half.

Every section opens with a **Rules defined here** line naming the ids it owns,
so you can confirm you opened the right file before reading it. A section
that says `none` states no law of its own — everything in it is cited.

## Routing

| Touching | Read |
|---|---|
| Structure, domains, DI, config placement, which library owns a concern | `01-project-structure.md` |
| Domain and transport schemas | `02-schemas.md` |
| HTTP controller, request/response contract, availability read for a unique field | `03-controllers.md` |
| Application operation, transaction or saga decision | `04-use-cases.md` |
| Persistence composition, soft delete, indexes, migrations | `05-repositories.md` |
| Entity guards and mutations | `06-entities.md` |
| External integration not covered by a lib | `07-adapters.md` |
| Consumer, queue handler or scheduled job | `08-background-handlers.md` |
| Domain event semantics | `09-events.md` |
| Error ownership and stable codes | `10-error-handling.md` |
| Authentication, authorization and secure input | `11-security.md` |
| Unit, integration and e2e tests | `12-testing.md` |
| Logs, metrics and operational signals | `13-telemetry.md` |
| Timeout, retry, circuit breaker, degradation of a dependency | `14-resilience.md` |
| Idempotency key, duplicate delivery, replay × skip | `15-idempotency.md` |

## Before you finish

1. **Scope.** Does every protected operation take its scope from the
   authenticated profile rather than from client input? (`ARC-SEC-1`)
2. **Boundary.** Is the controller/handler still thin — translate, validate
   shape, delegate — with no rule that leaked into it? (`ARC-DEL-1`)
3. **Write strategy.** Transaction, compensation or event: is it stated, and is
   the transaction closed before any external call? (`ARC-CON-1`, `ARC-CON-3`)
4. **Ownership.** Did you write infrastructure that a `@turystack/*` library
   already owns? (`01-project-structure.md` › *Which library owns which
   concern*, `ARC-LAY-8`)
5. **Errors.** Is every throw a catalogue class with a stable code, and does
   every catch either produce feedback or rethrow? (`ARC-ERR-3`, `ARC-ERR-7`)
6. **Async.** Does the handler you touched still work if the message arrives
   twice, or late? (`ARC-IDM-1`, `ARC-IDM-3`)
7. **Tests.** Does the change have the level of test that proves it — and does
   the data come from a factory rather than a literal? (`ARC-TST-1`,
   `ARC-TST-3`)
8. **Exposure.** Any secret or PII newly reachable in a response, log or span?
   (`ARC-SEC-7`)
9. **Uniqueness.** Does every unique constraint a consumer needs to anticipate
   have its own availability read, answering `200` with the same catalogue code
   the write throws? (`CTL-8`, `ARC-CON-4`)

Each id above carries a gate binding in its Invariants table, and
`turystack-proof` prints this same list with real pass/fail — `manual` bindings
stop for a person to sign. Binding kinds: `turystack-architecture-pattern` ›
`00-overview.md` › *Gate*.

## Ownership rule

This skill answers **what decision applies and where code belongs**. The library
answers **how its API is registered and called**. If this skill repeats a
library README, the library wins and the duplication should be removed.
