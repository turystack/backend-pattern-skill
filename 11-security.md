# Security

**Concept.** Layered defense: authenticate (who it is), authorize (what they can do, over which resource), validate/sanitize every input, protect sensitive data at rest and in transit, limit abuse (rate limit). Least privilege at every point.

> **How to read this file.** 🌐 Generic pattern is the portable law; 🛠️
> Project-specific is that same law expressed as code in TypeScript · NestJS ·
> @turystack · Drizzle · Zod. The split, `XXX-n` versus `XXX-Ln`, and why an
> `ARC-…` law is cited and never restated: `turystack-backend-pattern` › *How
> a section is written*. **This section owns no law of its own** — every rule
> here is constitutional, defined in `turystack-architecture-pattern` ›
> `07-security.md` and cited as `ARC-SEC-n`. That is not an accident: what
> makes an authorization check correct does not change when the framework
> does. What is genuinely backend-specific is the *mechanism*, and that is the
> whole 🛠️ half.

---

**Rules defined here:** none — every rule this file states is defined
elsewhere and cited by id.

## 🌐 Generic pattern (portable — stack-independent)

### Governed by the constitution

Read `07-security.md` in the constitution for the law's text. Here is the map
from each law to the backend's expression of it:

| ID | Law | How this stack expresses it |
|---|---|---|
| `ARC-SEC-1` | Scope and ownership come from the authenticated context. | `@AuthenticatedProfile()`; never `@Body`/`@Query`/`@Param` for scope |
| `ARC-SEC-2` | The backend is the authorization authority. | the guard decides; whatever the client renders is presentation |
| `ARC-SEC-3` | Authorization is operation **plus** resource. | `@ACL('invite:read', …)` with the workspace declared by the route |
| `ARC-SEC-4` | Input validated at the edge; an unknown field is rejected. | Zod `.strict()` on the route schema (see `02-schemas.md`) |
| `ARC-SEC-5` | Mass assignment impossible by construction. | `.pick` on the schema; nothing outside the pick reaches the use-case |
| `ARC-SEC-6` | A secret is hashed at rest and compared timing-safely. | `hashCode`/`verifyCode`; never `===` |
| `ARC-SEC-7` | Secrets and PII never enter a log, span, metric or response. | `schema.omit({ password: true })`; no secret field in a log |
| `ARC-SEC-8` | A third-party request has its origin verified before any effect. | webhook middleware validates the provider signature, rejects `401` |
| `ARC-SEC-9` | A sensitive operation is traceable. | the operation logs actor and time (see `13-telemetry.md`) |
| `ARC-SEC-10` | Untrusted data is neutralized at the **output** point. | bound SQL parameter, escaping per format, CSV formula prefix, log field |
| `ARC-SEC-11` | A credential has a single owner. | read through the owning service, never from storage |
| `ARC-SEC-12` | A permission id comes from the product's single catalogue. | ids born here and published through contract generation |

**Where the old local ids went.** This section used to restate seven of those
laws as `SEC-1`…`SEC-7`, with numbering that did not line up (`SEC-4` was
`ARC-SEC-3`, `SEC-7` was `ARC-SEC-4`). Anything citing the old ids maps as:
`SEC-1`→`ARC-SEC-1`, `SEC-2`→`ARC-SEC-6`, `SEC-3`→`ARC-SEC-7`,
`SEC-4`→`ARC-SEC-3`, `SEC-5`→`ARC-SEC-5`, `SEC-6`→`ARC-SEC-8`,
`SEC-7`→`ARC-SEC-4`.

`ARC-SEC-10` is not a frontend concern. The backend writes into several
interpreters: SQL (bound parameter, never concatenation), email and PDF
(escaping for the final format), CSV (formula prefix neutralized), shell (bound
argument), structured log (a field, never interpolation — `ARC-OBS-3`).
`ARC-SEC-4` validates the input; it does not know where the value goes next.

`ARC-SEC-12` puts the catalogue's owner here: permission identifiers are born on
this side and travel through contract generation (`ARC-CTR-1`). Renaming one
without publishing is a contract break, not an internal refactor.

---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · `@turystack/nestjs-iam`):**

- **ARC-SEC-1** — `@AuthenticatedProfile()` injects the `IamProfile` resolved by the guard; the use-case receives `profile.organizationId`/`workspaceId` **from the profile**, never from `@Body`/`@Query`/`@Param`, to define ownership. After authorizing, the entity confirms the resource belongs to the scope.
- **ARC-SEC-6** — `hashCode(plainCode)` (helper alongside the authentication domain) before persisting; `verifyCode(plain, hash)` (timing-safe) on verification; never `otpCode.code === plain`.
- **ARC-SEC-7** — `schema.omit({ password: true })` on the response schema; the log never receives `password`/`token`/PII fields (see `13-telemetry`). Secrets (e.g. `JWT_SECRET`) come from the validated env — never literals (see `01-project-structure`).
- **ARC-SEC-3** — `@ACL('invite:read')` covers the operation (401 without a token, 403 without permission — CASL); JWT tokens are issued **per workspace** (`IamTokenService.issueTokens(userId, { workspaceId })`), and workspace grants only hold in the workspace declared on the route via `@ACL(perm, (request) => ({ workspaceId: request.params.workspaceId }))`. `@Auth()` alone covers authentication only (401); the two stack up in a fully protected controller (see `03-controllers`).
- **ARC-SEC-5** — the DTO uses `.pick` on the Zod schema; whatever is not in the pick never reaches the use-case/repo (see `03-controllers`).
- **ARC-SEC-8** — the webhook middleware reads `X-Signature`/`X-Hub-Signature` and rejects with `401` before reaching the handler.
- **ARC-SEC-4** — Zod schema with `.strict()` on the route; lengths via `.max()`/`.min()`/`.regex` (see `02-schemas`).

### ✅ How to do it

**Scenario 1 — authorization by operation + scope + ownership (anti-IDOR):** `[ARC-SEC-1, ARC-SEC-3]`
```typescript
// operation via ACL; workspace declared by the route; ownership from the profile — never from the input
@ACL('invite:read', (request) => ({ workspaceId: request.params.workspaceId }))
@Get('workspaces/:workspaceId/invites/:inviteId')
async getInvite(
  @Param('inviteId') inviteId: string,
  @AuthenticatedProfile() profile: IamProfile,
) {
  return this.getInviteUseCase.execute({ inviteId, organizationId: profile.organizationId })
}

// inside the use-case's execute: confirm the resource belongs to the requester's scope
async execute(input: GetInviteInput) {
  const invite = await this.db.invites.findById(input.inviteId)
  invite.checkBelongsToOrganization(input.organizationId) // 403/404 if it belongs to another org
  return invite
}
```

**Scenario 2 — secret hashed before persisting; timing-safe comparison:** `[ARC-SEC-6]`
```typescript
// secret hashed before persisting; timing-safe comparison
await this.db.otpCodes.create({ ...input, code: await hashCode(plainCode) })
const matches = await verifyCode(plainCode, otpCode.code) // timing-safe, never ===
```

**Scenario 3 — response DTO omits the secret explicitly:** `[ARC-SEC-7]`
```typescript
// response DTO omits the secret
export const userResponse = userSchema.omit({ password: true })
```

### ❌ Never do

```typescript
// ❌ [ARC-SEC-6] persisting a secret (OTP/password/token) in plaintext
await this.db.otpCodes.create({ code: plainCode })

// ❌ [ARC-SEC-1] scope coming from client input (IDOR)
async listOrders(@Query('organizationId') organizationId: string) {}

// ❌ [ARC-SEC-1, ARC-SEC-3] authenticating only, without operation or scope/ownership (IDOR)
@Auth() // authenticates, but without @ACL + ownership one tenant sees another's resources

// ❌ [ARC-SEC-5] mass assignment: spreading raw input into the repo/entity
await this.db.users.create({ ...request.body }) // only fields via .pick in the DTO

// ❌ [ARC-SEC-7] leaking a secret in the response or in the log
return user // contains password
this.logger.info('signIn', { email, password })

// ❌ [ARC-SEC-6] comparing a secret with === (timing attack)
if (otpCode.code === plainCode) { /* ... */ }

// ❌ [ARC-SEC-8] processing a webhook without validating the provider's signature
@Post('webhook') async handleWebhook(@Body() body: unknown) { /* no verification */ }
```

### Adjacent concerns and who owns them

None of these is undecided — each already has an owner, and the reason they are
listed here is that a security review reaches for them and needs to know where
to look rather than inventing a local implementation (`ARC-LAY-8`):

| Concern | Owner | The law that applies |
|---|---|---|
| Rate limiting, lockout after repeated failure | `@turystack/nestjs-rate-limit` | `ARC-SEC-3` — abuse is an authorization outcome, answered with `429` from the catalogue |
| TLS, security headers, CORS | `@turystack/nestjs-server` configuration | `ARC-TOP-3` — declared by the app that terminates the request, not per route |
| Secret storage and rotation | `@turystack/nestjs-config` + the deployment | `ARC-SEC-11` — one owner per credential; rotation never means a second copy |
| Output sanitization per interpreter | the writing adapter | `ARC-SEC-10` — neutralized at the output point, per destination |
| Encryption at rest | the database/storage layer | `ARC-SEC-6` — a secret is hashed; a reversible field is encrypted, and the key has one owner |
| Audit trail of a sensitive operation | `@turystack/nestjs-logger` + the use-case | `ARC-SEC-9` — who ran it and when, as fields, never interpolated |
| Erasure and retention of what is stored here | the domain that owns the record | `ARC-DAT-1`, `ARC-DAT-3` |

What is genuinely open is **the numbers**: rate limits per surface, lockout
thresholds and key rotation cadence are product decisions, and they belong in
the app's configuration schema (`ARC-TOP-6`) where they are validated at boot —
not in this skill, and not as literals in a guard.
