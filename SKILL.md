---
name: tury-stack-backend-pattern
description: "Backend application constitution for Turystack projects. Defines boundaries, domain layers, transport contracts, use-cases, persistence policy, adapters, background handlers, events, errors, security, testing and telemetry. Libraries own setup and API documentation; this skill owns decisions and invariants."
---

# tury-stack-backend-pattern

Use this skill when creating, changing or reviewing backend application code.

## Prerequisite

`tury-stack-architecture-pattern` is the constitution and comes first. The law
lives there; this skill is how the backend expresses it. A law from the
constitution is never restated here — it is cited by id (`ARC-n`, `LAY-n`,
`CON-n`, `DEL-n`, `ERR-n`, `SEC-n`, `OBS-n`, `IDM-n`, `RES-n`, `TST-n`, `CTR-n`,
`TOP-n`). When the two disagree, the constitution wins.

## Reading order

0. Read `tury-stack-architecture-pattern` first.
1. Detect the project context in `00-overview.md`.
2. Read `01-project-structure.md`.
3. Read only the sections touched by the task.
4. Consult the installed `@turystack/*` library documentation for setup, API,
   decorators and adapter options. Never reconstruct library usage from this skill.

## Routing

| Touching | Read |
|---|---|
| Structure, domains, DI, config placement | `01-project-structure.md` |
| Domain and transport schemas | `02-schemas.md` |
| HTTP controller, request/response contract | `03-controllers.md` |
| Application operation, transaction or saga decision | `04-use-cases.md` |
| Persistence composition, soft delete, indexes | `05-repositories.md` |
| Entity guards and mutations | `06-entities.md` |
| External integration not covered by a lib | `07-adapters.md` |
| Consumer, queue handler or scheduled job | `08-background-handlers.md` |
| Domain event semantics | `09-events.md` |
| Error ownership and stable codes | `10-error-handling.md` |
| Authentication, authorization and secure input | `11-security.md` |
| Unit, integration and e2e tests | `12-testing.md` |
| Logs, metrics and operational signals | `13-telemetry.md` |

## Ownership rule

The skill answers **what decision applies and where code belongs**. The library
answers **how its API is registered and called**. If this skill repeats a library
README, the library wins and the duplication should be removed.
