# Security

**Concept.** Layered defense: authenticate (who it is), authorize (what they can do, over which resource), validate/sanitize every input, protect sensitive data at rest and in transit, limit abuse (rate limit). Least privilege at every point.

> **How to read this file.** Two parts, deliberately separated:
> - **🌐 Generic pattern** — the **portable law**. Holds in any stack (NestJS, PHP, etc.); the text stays true even after swapping frameworks. This is what review enforces as an invariant.
> - **🛠️ Project-specific** — the **code** that implements each rule in the current stack (TypeScript · NestJS · @turystack · Drizzle · Zod). Swapping stacks rewrites **only** this part; the law above does not change.
>
> Each rule carries an id `SEC-n` linking the law (generic) to the code (specific). Gates bind by **id**, not by file/line. `SEC-L*` rules are **stack lints**: they exist only because of the current language's ergonomics (they invert in another stack), but they stay id-registered and enforced in this project.

---

## 🌐 Generic pattern (portable — stack-independent)

**SEC-1 — auth-context scope/ownership (anti-IDOR).**

The requester's scope (organization, tenant, etc.) **always** comes from the auth-context (the authenticated identity), never from client input. Using the input as scope means any client can claim to belong to any tenant — classic IDOR. After authorizing the operation, the system confirms the resource belongs to the scope extracted from the auth-context. **[SEC-1]**

**SEC-2 — secret hashed at rest; timing-safe comparison.**

OTPs, passwords and tokens are **hashed/encrypted** before being persisted. The plaintext value exists only in memory, for immediate use (sending/verifying). Secret comparison uses a **timing-safe** function — never direct string comparison (`===`), which leaks time and enables a timing attack. **[SEC-2]**

**SEC-3 — secret never in a log nor in a response.**

Passwords, tokens and PII do not appear in log lines nor in response payloads. The response schema omits sensitive fields **explicitly** — never by accident or convention. (see 13-telemetry, 03-controllers) **[SEC-3]**

**SEC-4 — permission gating: operation + scope (least privilege).**

Every access to a protected resource requires an **operation** check (what the requester may do) **and** a **scope/ownership** check (over which resource). A global role alone is not enough: without scope, a requester sees/edits resources from other tenants. Grant the minimum privilege needed. **[SEC-4]**

**SEC-5 — no mass assignment.**

Raw client input is never spread directly into an entity/repo write. Only fields derived by explicit projection (pick/select) from the input reach persistence. (see 03-controllers) **[SEC-5]**

**SEC-6 — signature validation on webhooks.**

Webhook endpoints verify the **provider's signature** before processing any payload. A webhook without a valid signature is rejected immediately. **[SEC-6]**

**SEC-7 — input validated at the edge; unknown fields rejected.**

The schema validates everything at the edge (type, requiredness, length, range, enum); unknown fields are rejected (strict). (see 02-schemas, 03-controllers) **[SEC-7]**

### Invariants (the law the gates enforce)

> Bind by **id**. `constitutional` = portable invariant (holds in any stack). `stack lint` = exists only because of the current language's ergonomics and inverts in another stack — still id-registered and enforced here. The **detector** (how the gate catches the violation) is project-specific and lives in the 🛠️ part below.

| ID | Law (one line) | Class | Detector (🛠️) |
|---|---|---|---|
| SEC-2 | Secret hashed at rest; timing-safe comparison, never `===` | constitutional | Scenario 2 / ❌ |
| SEC-4 | Permission gating: operation + scope/ownership; a global role is not enough | constitutional | Scenario 1 / ❌ |
| SEC-6 | Webhooks validate the provider's signature before processing | constitutional | ❌ |

## Governed by the constitution

These laws live in `tury-stack-architecture-pattern` and are not restated here.
What follows in this section is how the Turystack backend expresses them.

| ID | Law |
|---|---|
| `ARC-SEC-1` | Scope from the authenticated context. |
| `ARC-SEC-2` | The backend is the authorization authority. |
| `ARC-SEC-3` | Operation plus resource. |
| `ARC-SEC-4` | Input validated at the edge. |
| `ARC-SEC-5` | Mass assignment impossible by construction. |
| `ARC-SEC-6` | Secret hashed and comparison timing-safe. |
| `ARC-10` | Secrets and PII out of logs, spans and responses. |
| `ARC-SEC-8` | Third-party origin verified. |
| `ARC-SEC-9` | A sensitive operation is traceable. |
| `ARC-SEC-10` | Untrusted data neutralized on **output**, per destination interpreter. |
| `ARC-SEC-11` | A credential has a single owner; the consumer asks, it does not fetch. |
| `ARC-SEC-12` | Permissions come from the product's single catalogue, published by this backend. |

`ARC-SEC-10` is not a frontend concern. The backend writes into several
interpreters: SQL (bound parameter, never concatenation), email and PDF (escaping
for the final format), CSV (formula prefix neutralized), shell (bound argument),
structured log (a field, never interpolation — `ARC-OBS-3`). `ARC-SEC-4`
validates the input; it does not know where the value goes next.

`ARC-SEC-12` puts the catalogue's owner here: permission identifiers are born on
this side and travel through contract generation (`ARC-CTR-1`). Renaming one
without publishing is a contract break, not an internal refactor.


---

## 🛠️ Project-specific (TypeScript · NestJS · @turystack · Drizzle · Zod)

> Code that implements the rules above in this stack. **Swapping stacks rewrites only this part.** Each block inherits the id of the rule it demonstrates.
>
> **⚠️ Comments in the examples are didactic** — they explain the rule being demonstrated. **Never copy a comment into the code**: the standard is zero comments (see Code Quality).

**Mechanisms per rule (NestJS · `@turystack/nestjs-iam`):**

- **SEC-1** — `@AuthenticatedProfile()` injects the `IamProfile` resolved by the guard; the use-case receives `profile.organizationId`/`workspaceId` **from the profile**, never from `@Body`/`@Query`/`@Param`, to define ownership. After authorizing, the entity confirms the resource belongs to the scope.
- **SEC-2** — `hashCode(plainCode)` (helper alongside the authentication domain) before persisting; `verifyCode(plain, hash)` (timing-safe) on verification; never `otpCode.code === plain`.
- **SEC-3** — `schema.omit({ password: true })` on the response schema; the log never receives `password`/`token`/PII fields (see `13-telemetry`). Secrets (e.g. `JWT_SECRET`) come from the validated env — never literals (see `01-project-structure`).
- **SEC-4** — `@ACL('invite:read')` covers the operation (401 without a token, 403 without permission — CASL); JWT tokens are issued **per workspace** (`IamTokenService.issueTokens(userId, { workspaceId })`), and workspace grants only hold in the workspace declared on the route via `@ACL(perm, (request) => ({ workspaceId: request.params.workspaceId }))`. `@Auth()` alone covers authentication only (401); the two stack up in a fully protected controller (see `03-controllers`).
- **SEC-5** — the DTO uses `.pick` on the Zod schema; whatever is not in the pick never reaches the use-case/repo (see `03-controllers`).
- **SEC-6** — the webhook middleware reads `X-Signature`/`X-Hub-Signature` and rejects with `401` before reaching the handler.
- **SEC-7** — Zod schema with `.strict()` on the route; lengths via `.max()`/`.min()`/`.regex` (see `02-schemas`).

### ✅ How to do it

**Scenario 1 — authorization by operation + scope + ownership (anti-IDOR):** `[SEC-1, SEC-4]`
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

**Scenario 2 — secret hashed before persisting; timing-safe comparison:** `[SEC-2]`
```typescript
// secret hashed before persisting; timing-safe comparison
await this.db.otpCodes.create({ ...input, code: await hashCode(plainCode) })
const matches = await verifyCode(plainCode, otpCode.code) // timing-safe, never ===
```

**Scenario 3 — response DTO omits the secret explicitly:** `[SEC-3]`
```typescript
// response DTO omits the secret
export const userResponse = userSchema.omit({ password: true })
```

### ❌ Never do

```typescript
// ❌ [SEC-2] persisting a secret (OTP/password/token) in plaintext
await this.db.otpCodes.create({ code: plainCode })

// ❌ [SEC-1] scope coming from client input (IDOR)
async listOrders(@Query('organizationId') organizationId: string) {}

// ❌ [SEC-1, SEC-4] authenticating only, without operation or scope/ownership (IDOR)
@Auth() // authenticates, but without @ACL + ownership one tenant sees another's resources

// ❌ [SEC-5] mass assignment: spreading raw input into the repo/entity
await this.db.users.create({ ...request.body }) // only fields via .pick in the DTO

// ❌ [SEC-3] leaking a secret in the response or in the log
return user // contains password
this.logger.info('signIn', { email, password })

// ❌ [SEC-2] comparing a secret with === (timing attack)
if (otpCode.code === plainCode) { /* ... */ }

// ❌ [SEC-6] processing a webhook without validating the provider's signature
@Post('webhook') async handleWebhook(@Body() body: unknown) { /* no verification */ }
```

🔵 **to provide:** Rate Limiting (`@turystack/nestjs-rate-limit`) / lockout · Additional sanitization · Encryption at rest/in transit (TLS) · secret rotation · security headers/CORS.
