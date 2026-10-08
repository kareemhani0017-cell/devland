# DevLand Enterprise — Phase 1, Part 3: Access, security, authentication, audit

Covers deliverables 7, 11, 12, 13.

## 1. Permission model (RBAC + ABAC)

### 1.1 Concepts
- **Permission** = a string key from a code-defined catalog (`packages/contracts/permissions.ts`), seeded into `Permission`. Format `domain.resource.action`. `isSensitive` keys require step-up 2FA (dashboard) and/or an approval.
- **AccessRole** = DevLand's internal role. The Owner maps an **existing Discord role ID** to it in the dashboard. *The bot never creates roles.* Roles can inherit (`parentRoleId`).
- **ABAC conditions** = optional JSON condition tree on a role-permission or grant, evaluated against attributes.
- **AccessGrant** = per-user override, temporary, JIT or delegated access (mandatory reason; expiry mandatory for temporary/JIT).

### 1.2 Attributes available to conditions
| Group | Attributes |
|---|---|
| subject | `userId`, `roles[]`, `customerLevel`, `developerId`, `verificationLevel`, `assuranceLevel` (session 2FA state) |
| resource | `type`, `id`, `serviceId`, `status`, `paymentStatus`, `amountEgp` (EGP-equivalent), `customerId`, `assignedDeveloperId`, `channelId` |
| action | permission key |
| environment | `now`, `source` (`discord`\|`dashboard`\|`api`), `ipRisk`, `lockdown` |

Condition DSL (stored as JSON; same shape later powers the Security Rule Engine):
```json
{ "all": [
  { "attr": "resource.serviceId", "op": "in", "value": "$subject.poolServiceIds" },
  { "attr": "resource.assignedDeveloperId", "op": "isNull" },
  { "any": [ { "attr": "resource.amountEgp", "op": "lte", "value": 5000 },
             { "attr": "subject.assuranceLevel", "op": "gte", "value": 2 } ] }
] }
```

### 1.3 Evaluation algorithm (`AuthorizationService.can`) — default deny
1. **Global gates:** lockdown mode on and action not on the lockdown allowlist → DENY. Feature-flag kill switch for the action → DENY.
2. **Resolve subject:** user → mapped `AccessRole`s (from Discord member roles, cached 60 s, invalidated on `GuildMemberUpdate`) + inherited roles + active, unexpired `AccessGrant`s.
3. **Collect rules** for the permission key: role rules and grants, each ALLOW/DENY with optional conditions.
4. **Evaluate conditions** against attributes; discard rules whose conditions are false.
5. **Decision:** any matching DENY → DENY (deny overrides). Else any matching ALLOW → ALLOW. Else DENY.
6. **Sensitivity:** if allowed and `Permission.isSensitive`: require `session.stepUpAt` within N minutes (dashboard) or an approved `ApprovalRequest`/active JIT grant; otherwise return `STEP_UP_REQUIRED` / `APPROVAL_REQUIRED`.
7. **Field projection:** the decision carries a *visibility profile* used by serializers (see 1.5). Authorization is checked in the service layer, so Discord, API and workers get identical outcomes.
8. **Audit:** denials of sensitive actions, all grants/overrides, and all sensitive allows are audited with the decision trace.

### 1.4 Permission catalog (MVP subset; full catalog grows per phase)

| Key | Sensitive | Typical holders (Owner configures) |
|---|---|---|
| `setup.run` | yes | Owner |
| `settings.read` / `settings.write` | write: yes | Owner |
| `exchange_rate.write` | yes | Owner |
| `role_mapping.manage` | yes | Owner |
| `store.service.manage`, `store.feature.manage` | no | Owner |
| `order.create` | no | Customer (implicit) |
| `order.read.own` | no | Customer (implicit; ABAC `resource.customerId == subject.userId`) |
| `order.read.pool` | no | Developers (ABAC: unassigned in own pool OR assigned to self) |
| `order.read.any` | no | Owner, Support |
| `order.claim` | no | Developers (ABAC: pool match, under open-order limit) |
| `order.assign` | yes | Owner |
| `order.transition.*` (per transition) | cancel: yes | per transition table (doc 04 §6) |
| `developer.request.decide` | no | Owner |
| `deadline.propose` | no | Assigned developer |
| `deadline.decide` | no | Customer (implicit, own order) |
| `deadline.extend` | yes | Owner only |
| `delivery.submit` | no | Assigned developer |
| `delivery.respond` | no | Customer (implicit, own order) |
| `change_request.override_limit` | yes | Owner |
| `payment.method.manage` | yes | Owner |
| `payment.proof.submit` | no | Customer (own order) |
| `payment.proof.view` | yes | **Owner only by default** (+ customer sees own via `.own`) |
| `payment.review` | yes | Owner (optionally Support with amount ABAC) |
| `payment.status.view` | no | Developers, Support, Owner |
| `finance.commission.view.own` | no | Developer |
| `finance.commission.view.all` | yes | Owner |
| `developer.balance.view.own` | no | Developer |
| `withdrawal.request` | no | Developer |
| `withdrawal.review` | yes | Owner |
| `review.submit` | no | Customer, own completed order |
| `customer.read.pii` | yes | Owner |
| `audit.read` / `audit.export` | yes | Owner |
| `dashboard.access` | no | Owner (+ delegated staff later) |
| `lockdown.toggle` | yes | Owner |

### 1.5 Visibility profiles (field-level rules)
Serializers take a profile, never the raw row.

| Profile | Sees on an Order | Never sees |
|---|---|---|
| `customer` | own order, price, discount, own payment info/proof, developer display name, deadline, deliveries, reviews | commission split, other customers, private developer notes, owner notes, fraud signals |
| `developer` (assigned) | requirements, features, files/links, deadline, customer **display name + Discord mention**, `paymentStatus`, own commission (total, DevLand %, dev %, dev amount) | payment proofs, payment account details, customer email/phone/verification data, owner notes, risk data, other developers' finances |
| `developer` (pool, unclaimed) | service, feature list, budget summary, requirements needed to decide | customer identity beyond what policy allows |
| `support` | order/ticket context, `paymentStatus` | proofs (unless granted), commission |
| `owner` | everything | — |

## 2. Security architecture (deliverable 11)

Layered, with a clear statement of what is enforced where.

| Layer | Controls |
|---|---|
| Edge | TLS everywhere; `helmet` headers (CSP, HSTS, frame-ancestors none, no-sniff, referrer-policy); strict CORS allowlist = dashboard origin only |
| Input | Every request/interaction payload validated by `zod` schema from `packages/contracts`; Discord modal/select values treated as untrusted input; size limits; file type sniffing (magic bytes) and max size for uploads |
| AuthN | Dashboard: Discord OAuth2 + server-side sessions (§3). Bot: Discord signature/gateway identity → resolved `User`. API keys/OAuth2 apps: later phase |
| AuthZ | `AuthorizationService` only; controllers/handlers declare `@RequirePermission('x')`; no inline role checks. Service layer re-checks for non-HTTP callers |
| CSRF/XSS | `SameSite=Lax` HttpOnly Secure cookie + `Origin`/`Sec-Fetch-Site` check + double-submit token on mutating requests; React escaping, no `dangerouslySetInnerHTML` without sanitizer; CSP without `unsafe-inline` (nonces) |
| Rate limiting | Redis sliding-window per user, per IP, per endpoint, per Discord interaction type (button spam), with burst control; limit breaches write `SecurityEvent`. Risk-adaptive limits and abuse cases are a later phase on the same interface |
| Data protection | Field-level AES-256-GCM for contact info, payment account details, payout details, TOTP secrets (`enc:v1:<keyId>:<iv>:<tag>:<ct>`); encrypted files; Postgres/disk encryption is the host's — verified/reported, not assumed |
| Secrets | `SecretProvider` interface; `env` driver in Phase 2; no secrets in Git (gitleaks in CI + push protection); separate secrets per environment. Vault/dynamic secrets only when a vault exists |
| Keys & rotation | Keyring (`CRYPTO_KEYS`: id→key). New data uses the active key; old keys stay for decrypt; `pnpm rotate-keys` re-encrypts. **Rotation is manual-triggered with the env driver; automatic rotation requires an external KMS and is not claimed** |
| Integrity | State machines + DB constraints + idempotency (doc 02); audit chain (§4) |
| Errors | Customers get a safe message + `errorId`; stack traces only in internal logs with `errorId`, `correlationId`, service, severity |
| Supply chain | lockfile, Renovate/Dependabot, `pnpm audit`, CODEOWNERS on `packages/db`, `packages/core/auth|access|audit|payments` |
| Discord-specific | Bot token only in secret store; least-privilege bot permissions & intents (no Message Content unless later needed); ticket channels permission-overwritten to customer + assigned developer + Owner; owner-only channels for proofs; DM failures handled gracefully |
| Lockdown | `system.lockdown` flag (DB + Redis): freezes order creation, payments, withdrawals, sensitive workers; invalidates sessions; Owner recovery path stays up (full spec in later phase, hook in `AuthorizationService` step 1 exists from day one) |

**Threat model summary (STRIDE-lite, MVP):**
- *Spoofed customer/developer in Discord* → identity = Discord-signed interaction user ID; role checks from Discord member data, never from user-supplied text.
- *Forged payment proof / reuse of one screenshot* → SHA-256 duplicate detection, mandatory Owner review, amount/currency/method shown next to proof.
- *Double approve / double credit / double delivery acceptance* → unique partial indexes + idempotency keys + `FOR UPDATE`.
- *Developer sees customer payment data* → visibility profiles + proof permission is Owner-only by default.
- *Owner account takeover* → 2FA, step-up for sensitive actions, short idle timeout, device list, recovery codes (full recovery flow later).
- *Tampered history* → hash-chained audit + signed checkpoints + periodic verification job.
- *Redis loss* → reconcilers rebuild timers from Postgres.

## 3. Authentication & session architecture (deliverable 12)

### 3.1 Dashboard login
1. `GET /auth/discord` → redirect to Discord OAuth2 (`identify` scope; `state` + PKCE).
2. Callback: exchange code, fetch Discord user, upsert `User`.
3. **Owner bootstrap:** the first login whose `discordId == OWNER_DISCORD_ID` (env) gets the `owner` role. All later roles come from Discord-role → `AccessRole` mapping. Anyone without `dashboard.access` is rejected after login (and a `SecurityEvent` is written).
4. **2FA (TOTP via `otplib`)** is mandatory for any user with a sensitive permission. First Owner login forces enrolment; recovery codes are generated once (stored only as hashes). Passkeys/WebAuthn are a later enhancement behind the same `assuranceLevel` model.
5. Session creation: random 256-bit token → cookie `__Host-dl_session` (HttpOnly, Secure, SameSite=Lax, Path=/); DB stores only `SHA-256(token)`.

### 3.2 Session rules
| Property | Default |
|---|---|
| Idle timeout | 30 min (Owner/sensitive roles), configurable |
| Absolute lifetime | 12 h |
| Step-up window for sensitive actions | 5 min since last TOTP |
| Concurrent sessions | listed in dashboard, individually revocable, "logout all" |
| Fingerprint | `ipHash` + UA hash stored as **soft signals**; change → raise `riskScore`, may force step-up; never auto-trusted |
| Forced logout | Owner/lockdown/security rule sets `revokedAt` + reason; Redis cache entry deleted |
| Lookup | Redis cache (short TTL) → fallback to Postgres; revocation checks always honor DB |

IPs are stored hashed (HMAC with a server pepper) unless an investigation policy requires raw IPs — a privacy-by-default choice; flip via config if you need raw IPs for forensics.

### 3.3 Discord (bot) identity
No session. Each interaction resolves `User` by `interaction.user.id` and roles by `interaction.member.roles` (guild-verified by Discord). `SessionContext`-equivalent `ActorContext {userId, roles, source:'discord', correlationId}` is passed to services. Sensitive Owner actions in Discord (approve payment/refund/withdrawal) require a confirmation modal; actions above a configured amount direct the Owner to the dashboard for step-up 2FA instead (so Discord never becomes a 2FA bypass).

## 4. Audit / forensic architecture (deliverable 13)

### 4.1 What is recorded
Every state-changing business action produces an `AuditEvent` **inside the same transaction** as the change: who (`actorType`,`actorId`), what (`action`), target (`targetType`,`targetId`), when, why (`reason`), `before`/`after` snapshots (redacted of secrets/PII per field policy), `correlationId`, `requestId`, `ipHash`/`deviceHash`, `schemaVersion`.

### 4.2 Hash chain
```
hash(n) = SHA-256( canonicalJSON(event fields n, excluding hash) || hash(n-1) )
```
- `sequence` is gapless per `chainId`, assigned inside the transaction under `pg_advisory_xact_lock(chainKey)` so concurrent writers serialize and the chain stays linear. (Throughput cost is acceptable at DevLand's scale; chains can be partitioned per domain later via `chainId` without schema change.)
- Canonicalization uses a deterministic JSON serializer (sorted keys, no whitespace, UTC ISO timestamps) — versioned by `schemaVersion`.
- **Checkpoints:** a worker job every N minutes writes `AuditCheckpoint{upToSequence, hash, signature}` where `signature = HMAC(auditSigningKey, chainId||sequence||hash)`; the signing key is held **separately from the database** (secret provider). Someone with only DB access cannot forge a consistent chain + valid checkpoints.
- **Verification:** `audit-verify` job recomputes the chain between checkpoints, verifies signatures, writes `AuditVerificationRun`, and on mismatch emits `audit.integrity_violation_detected` (→ Discord alert to Owner, security event, later an incident).
- **Export:** evidence package = events + their hashes + checkpoints + verification report, signed.

### 4.3 What we can honestly claim
| Claim | True when |
|---|---|
| "Tamper-evident" | Always (chain + signed checkpoints + verification job) |
| "Append-only at DB level" | App role has no UPDATE/DELETE on `AuditEvent` (REVOKE) — otherwise only the trigger guards it, which a DB superuser could drop; `doctor` reports which applies |
| "Immutable / WORM" | **Only** if checkpoints/exports are anchored in storage with object lock (`capabilities.objectLock`) or another external anchor (`AuditCheckpoint.anchoredRef`). Until then the docs and dashboard say "tamper-evident", never "immutable" |
| "Cryptographically verifiable" | Yes, via recomputation + HMAC signature (symmetric: verifiable by whoever holds the signing key; asymmetric signing is a possible later upgrade) |

### 4.4 Audit coverage rules
- Writing audit is part of `UnitOfWork.commit()`; a mutation without an audit/outbox call fails an architecture test in CI ("every command handler emits ≥1 audit event").
- Reads of sensitive data (payment proof view, PII view, audit export) are audited too.
- Retention: audit rows are never deleted by app code; archival (move old partitions to cold storage with their checkpoints) is a later-phase policy.
