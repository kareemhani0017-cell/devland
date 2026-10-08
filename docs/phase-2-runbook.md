# DevLand — Phase 2 Runbook (execute with Claude Code on the real repository)

Purpose: turn the approved Phase 1 design into a **validated foundation** — workspace, schema + migration, config, infra adapters, crypto, queues, storage, audit, events, access, auth, bot foundation, dashboard shell, health, shutdown, CI, production build and deployment artifacts.

**Nothing in this runbook has been executed.** The authoring sandbox had no pnpm, Prisma, Docker, PostgreSQL or npm registry access. Every "expected result" below is a pass criterion to check, not a claim that it passed.

---

## 0. Ground rules for Claude Code (read first)

1. **Execute steps in order.** Do not start step N+1 until step N's "Expected result" is met and its tests pass. If a step fails, fix it inside that step; if a Phase 1 decision must change, write an ADR in `docs/adr/` and tell the owner instead of silently deviating.
2. **Never claim a command passed unless you ran it.** After each step append to `docs/phase-2-report.md`: command, exit code, trimmed output, date, git SHA. Skipped or failed items are recorded as skipped or failed.
3. **One commit per step** (`feat(step-07): database adapter`), on a feature branch; CI must be green before merging.
4. **No secrets in the repo, ever.** Only `.env.example` files with empty/dummy values. The Discord token pasted earlier in chat must be reset in the Discord Developer Portal before any bot step; use separate Discord applications for dev, staging and prod.
5. **No business domains yet** (orders, store, payments logic, tickets, delivery, reviews). Phase 2 is foundation only. Stop after step 28 and report; do not start Phase 3.
6. **No WispByte-specific code.** WispByte stays behind adapters/config until its capabilities are verified (see `docs/deployment/wispbyte.md` §1).
7. **Decisions to confirm in step 2** (defaults chosen; record in `docs/adr/0002-decisions.md`):

| ID | Decision | Default | Why |
|---|---|---|---|
| D1 | Module format — **APPROVED** | Backend apps + shared packages compile with `tsc` to **CommonJS** + `.d.ts` in `dist/`; `packages/ui` is consumed by the dashboard **from source** via Next `transpilePackages` | NestJS is smoothest on CJS; Tailwind must scan `ui` sources anyway |
| D2 | Dashboard session cookie — **APPROVED** | Dashboard talks to API through a **same-origin BFF proxy** (Next rewrites/route handlers), so `__Host-dl_session` is host-only on the dashboard origin | `__Host-` cookies cannot span `dashboard.` and `api.` subdomains. Refines Phase 1 doc 03 §3.1 |
| D3 | Prisma client — **APPROVED** | Generated **as part of the production build/artifact** into `packages/db/src/generated` and compiled into `dist`. **`binaryTargets` is NOT set** (default/native only) until the real WispByte host OS/OpenSSL target is verified; artifacts built before that are **provisional and must not be deployed** | Production artifacts must not need the Prisma CLI or dev deps; a client generated for the wrong platform fails at first query |
| D4 | Node version — **DEFERRED, do not guess** | Verify the Node versions WispByte actually supports (wispbyte.md §1 row 1) **before** writing `.nvmrc`, `engines` or CI `node-version`. Until verified, local work uses whatever Node is installed, recorded in the report as "dev-only, not authoritative" | Avoid runtime drift |
| D5 | Deploy packaging — **APPROVED, verify before final** | `pnpm --filter <app> deploy --prod` into `artifacts/<app>`. Record `pnpm -v` and `pnpm deploy --help`; newer majors may require `inject-workspace-packages=true` or a legacy flag. Artifacts are **not final** until verified in the real environment (step 27) | Gives each service a self-contained prod-only folder |

---

## 0a. Owner-approved hard constraints (non-negotiable; each has a test)

1. **No unexecuted claims.** Only record results from commands actually run.
2. **No Phase 3, no business domains** (store, orders, tickets, payments logic, delivery, reviews).
3. **All WispByte capability assumptions must be verified before any production deployment** (wispbyte.md §1 fully filled with evidence). Until then artifacts are provisional.
4. **WispByte-specific functionality stays behind adapters.** No `wispbyte` driver or code until capabilities are confirmed.
5. **Do not set Prisma `binaryTargets`** until the host OS/OpenSSL environment is known.
6. **PostgreSQL advisory locks must never run through transaction-mode pooling.**
   - Add env `DB_POOLER_MODE` = `none | session | transaction` (default `none`) and optional `DATABASE_LOCK_URL` (direct or session-mode connection).
   - A `LockConnectionProvider` in `packages/db` opens a **dedicated direct/session connection** for `pg_advisory_xact_lock` and `pg_try_advisory_lock` (audit writer, outbox-relay leader, any future lock). It uses `DATABASE_LOCK_URL` if set, else `DIRECT_DATABASE_URL`/`DATABASE_URL` **only if** `DB_POOLER_MODE != transaction`.
   - If `DB_POOLER_MODE=transaction` and no `DATABASE_LOCK_URL`: **refuse to boot** (config validation, step 8). `doctor` additionally proves lock semantics (two sessions contend; second blocks/fails) against the real connection path.
   - Tests: boot refusal case; two-session contention test; audit chain stays gapless under concurrency using the lock connection.
7. **No assumed shared disk between API and Worker.** `STORAGE_DRIVER=local` in production is refused unless `STORAGE_LOCAL_SHARED_VERIFIED=true` is set deliberately after verifying shared persistent storage (§1 row 10 + a cross-process write/read test). Default `false`. `local` is always allowed for development and CI.
8. **Redis `noeviction` must be verified before relying on it.** At boot and in `doctor`, read `CONFIG GET maxmemory-policy`; production refuses to start BullMQ consumers unless it is `noeviction`. If the provider blocks `CONFIG`, production requires explicit `REDIS_NOEVICTION_VERIFIED=true` (set only after checking provider settings/docs). Correctness never depends on Redis: Postgres outbox + reconcilers remain the source of truth.
9. **PostgreSQL is the source of truth; Redis loss must not lose business data** (test in steps 11 and 15: Redis stopped, business transaction + outbox row still commit, delivery resumes after recovery).

---

## Final folder tree (approved; `packages/ui` is a dashboard-bundled package)

```
devland/
├─ apps/
│  ├─ api/        package.json  Dockerfile  .env.example  src/{main.ts,health/,shutdown.ts,http/}
│  ├─ bot/        package.json  Dockerfile  .env.example  src/{main.ts,gateway/,interactions/,renderers/,health/}
│  ├─ worker/     package.json  Dockerfile  .env.example  src/{main.ts,queues/,processors/,reconcilers/,health/}
│  └─ dashboard/  package.json  .env.example  next.config.ts  src/{app/,components/,features/,lib/}
├─ packages/
│  ├─ types/  contracts/  config/  db/  infra/  storage/  payments/
│  ├─ audit/  events/  access/  auth/  core/  i18n/  ui/  testing/  tooling/
├─ infrastructure/
│  ├─ docker/ (compose.dev.yml, compose.test.yml)
│  ├─ migrations/ (raw SQL + runner notes)   ├─ deployment/ (artifact manifests, templates)
│  ├─ monitoring/ (alert rules, dashboards)   └─ scripts/ (build-artifact, doctor, verify-audit-chain, events-replay, rotate-keys)
├─ docs/ (01..05, phase-2-runbook.md, phase-2-report.md, deployment/{wispbyte,vercel}.md, adr/)
├─ .github/{workflows/,CODEOWNERS}
└─ .gitignore  .dockerignore  .npmrc  .nvmrc  package.json  pnpm-workspace.yaml  turbo.json  CLAUDE.md
```

Dependency direction (enforced in step 3): `types → contracts, config → infra, storage, payments, db → audit, events, access, auth → core → apps`. `ui`, `i18n` are leaf packages; backend apps never import `ui`; dashboard never imports `db`, `core`, `auth`, `access`, `audit`, `events`, `storage`, `payments`.

## Deployment mapping (summary)

| Unit | Target | Artifact | Deployed? |
|---|---|---|---|
| `apps/api` | WispByte (adapter-agnostic) | `artifacts/api` (dist + prod node_modules) | Yes |
| `apps/bot` | WispByte | `artifacts/bot` | Yes (singleton) |
| `apps/worker` | WispByte | `artifacts/worker` | Yes (≥1) |
| `apps/dashboard` | Vercel | built by Vercel; `packages/ui`, `i18n`, `contracts`, `types` bundled into it | Yes |
| PostgreSQL / Redis / Storage | provider adapters (host TBD) | n/a | External |
| `docs/`, tests, `testing`, `tooling`, `infrastructure/monitoring`, `.github`, `.git`, any `.env*` | — | — | **Never deployed** |

---

## Step template

Each step lists: **Create/modify · Dependencies · Commands · Expected · Fails if · Fix before continuing · Tests required · Scope** (`local`, `CI`, `WispByte`, `Vercel`).

---

## Step 1 — Environment/tool prerequisites
- **Create/modify:** `docs/adr/0001-toolchain.md`; set root `packageManager` field. **Do NOT create `.nvmrc`/`engines`/CI `node-version` yet (D4 deferred):** first verify WispByte's supported Node version(s) (wispbyte.md §1 row 1) and record the evidence; only then pin them everywhere.
- **Dependencies:** Node LTS (D4), Corepack-enabled pnpm, Git, Docker + Compose (local only), GitHub repo with branch protection, a Discord *development* application + test guild, fresh bot token (old one reset).
- **Commands:** `node -v` · `corepack enable && pnpm -v` · `docker compose version` · `git --version`
- **Expected:** versions printed and recorded; Docker daemon reachable. Node pin happens only after WispByte's version is verified (otherwise recorded as "dev-only, not authoritative").
- **Fails if:** any tool missing, Docker not running, or a Node version is pinned without verification evidence.
- **Fix before continuing:** install/upgrade; no step may proceed on mismatched Node.
- **Tests required:** none (record versions in `phase-2-report.md`).
- **Scope:** local.

## Step 2 — pnpm + Turborepo workspace
- **Create/modify:** root `package.json` (scripts: `build typecheck lint test dev doctor`), `pnpm-workspace.yaml` (`apps/*`, `packages/*`), `turbo.json` (tasks `build`→`^build`, `typecheck`, `lint`, `test`, `dev` persistent), `.npmrc`, `.gitignore`, `.dockerignore`, `CLAUDE.md` (from doc 05 §5), `docs/adr/0002-decisions.md` (D1–D5). Scaffold every `apps/*` and `packages/*` with `package.json` (`@devland/<name>`, `"private": true`, `exports` → `dist`), `tsconfig.json`, empty `src/index.ts`. Internal deps via `workspace:*`.
- **Dependencies:** `turbo`, `typescript`.
- **Commands:** `pnpm install` · `pnpm turbo run build` · `pnpm turbo run typecheck`
- **Expected:** install succeeds with a committed `pnpm-lock.yaml`; every empty package builds; Turbo cache hits on a second run.
- **Fails if:** circular workspace deps, unresolved `workspace:*`, missing `exports`.
- **Fix before continuing:** correct package graph so it matches the dependency direction above.
- **Tests required:** a script `infrastructure/scripts/check-workspace.ts` asserting each package has name/exports/tsconfig (run in CI).
- **Scope:** local + CI.

## Step 3 — TypeScript / ESLint / Prettier / dependency-cruiser
- **Create/modify:** `packages/tooling/{tsconfig.base.json,tsconfig.node.json,tsconfig.next.json,eslint.config.*,prettier.config.*,.dependency-cruiser.cjs}`; strict TS (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`); ESLint rules: no `any`, no floating promises, no direct `process.env` outside `packages/config`, no `console` in services, no `prisma` import outside `packages/db` and repositories; dependency-cruiser rules encoding the layer diagram and the dashboard/bot restrictions.
- **Dependencies:** `eslint`, `typescript-eslint`, `prettier`, `dependency-cruiser`, `husky`, `lint-staged`.
- **Commands:** `pnpm lint` · `pnpm depcruise` (add script) · `pnpm format --check`
- **Expected:** all pass on the empty scaffold; a deliberate violation fixture (`packages/tooling/fixtures/bad-import`) makes `depcruise` exit non-zero.
- **Fails if:** rules too loose (fixture passes) or too strict (scaffold fails).
- **Fix before continuing:** adjust rules; keep the fixture test.
- **Tests required:** test that depcruise fails on the fixture; test that an `process.env` use outside config fails lint.
- **Scope:** local + CI.

## Step 4 — Prisma schema validation
- **Create/modify:** copy `schema.prisma` to `packages/db/prisma/schema.prisma`; `packages/db/package.json` scripts (`prisma:validate`, `prisma:format`, `prisma:generate`, `prisma:migrate:dev`, `prisma:migrate:deploy`); set generator `output` per D3; dummy `DATABASE_URL`/`DIRECT_DATABASE_URL` for validation only.
- **Dependencies:** `prisma`, `@prisma/client`. **Check the installed Prisma major:** the schema uses 6.x datasource syntax; if you are on 7.x move `url` into `prisma.config.ts` (ADR).
- **Commands:** `pnpm --filter @devland/db exec prisma validate`
- **Expected:** "The schema is valid". Record the exact output.
- **Fails if:** any validation error. Likely places to look: missing or ambiguous back-relations, relations needing explicit names, enum member reuse across enums, `@@unique` including nullable columns, Prisma-version syntax differences.
- **Fix before continuing:** edit only the schema; log each change and why in `docs/adr/0003-schema-fixes.md`; re-run until valid. Do not weaken constraints to silence errors.
- **Tests required:** CI job `prisma validate` (step 26).
- **Scope:** local + CI.

## Step 5 — Prisma format
- **Create/modify:** formatted `schema.prisma` (separate commit: formatting only).
- **Commands:** `pnpm --filter @devland/db exec prisma format` then `git diff --stat`; run again.
- **Expected:** second run produces no diff (idempotent).
- **Fails if:** formatting changes on every run, or semantics changed (review diff).
- **Fix before continuing:** commit the formatted file; add CI check (`prisma format` + `git diff --exit-code`, or the CLI's check flag if your version has one).
- **Tests required:** CI formatting check.
- **Scope:** local + CI.

## Step 6 — Initial migration (+ raw SQL constraints)
- **Create/modify:** `infrastructure/docker/compose.dev.yml` (Postgres, Redis, MinIO, Mailpit; named volumes; no host secrets), `packages/db/prisma/migrations/<ts>_init/`, `packages/db/prisma/migrations/<ts>_constraints/migration.sql` (partial unique indexes, CHECKs, append-only triggers from doc 02 §4), `infrastructure/migrations/README.md`.
- **Dependencies:** Docker Postgres (version equal to your target host; unverified for WispByte).
- **Commands:**
  1. `docker compose -f infrastructure/docker/compose.dev.yml up -d postgres`
  2. `pnpm --filter @devland/db exec prisma migrate dev --name init --create-only` → **read the generated SQL**
  3. create the constraints migration (`--create-only`, paste SQL)
  4. `pnpm --filter @devland/db exec prisma migrate dev`
  5. prove clean apply: `prisma migrate reset --force --skip-seed`
  6. prove no drift: `prisma migrate diff --from-migrations prisma/migrations --to-schema-datamodel prisma/schema.prisma --shadow-database-url <shadow> --exit-code`
- **Expected:** migrations apply on an empty DB; reset+apply works; drift check exits 0.
- **Fails if:** SQL errors (the doc 02 snippets use unquoted camelCase columns — **they must be double-quoted to match Prisma's generated names**), drift exits non-zero, trigger/index missing.
- **Fix before continuing:** correct raw SQL against `\d "TableName"`; never edit an applied migration — add a new one.
- **Tests required (integration, real Postgres):** inserting two `APPROVED` payments for one order fails; second active assignment fails; second open support ticket per customer fails; second accepted delivery fails; `UPDATE`/`DELETE` on `AuditEvent`, `OrderStatusHistory`, `DeveloperLedgerEntry`, `WalletTransaction`, `ExchangeRate` raise; CHECK constraints reject bad values.
- **Scope:** local + CI.

## Step 7 — Database adapter (`packages/db`)
- **Create/modify:** `src/{client.ts,unit-of-work.ts,sequence.ts,locks.ts,optimistic.ts,money-mapper.ts,index.ts}`; `Money` value object in `packages/types` (Decimal-based, currency-explicit, no float ops); `UnitOfWork.run(fn, {correlationId})` (the only place `$transaction` is called); `allocateSequence(tx,key)` via `UPDATE … RETURNING`; `advisoryLock(tx,key)`; `updateWithVersion()`.
- **Dependencies:** `@prisma/client`, `decimal.js` (or Prisma's Decimal), `pg` (if raw helpers need it).
- **Commands:** `pnpm --filter @devland/db test` (Testcontainers or compose Postgres).
- **Expected:** tests pass; client builds to `dist` with generated client included.
- **Fails if:** Prisma client missing in `dist`, sequences skip/duplicate under concurrency.
- **Fix before continuing:** fix packaging (D3) and locking.
- **Tests required:** 100 parallel `allocateSequence` → unique, gapless; optimistic update conflict throws `ConcurrentModificationError`; rollback leaves no partial rows; Money add/split/round golden cases (e.g. commission splits sum exactly to total); Money rejects mixed currencies.
- **Scope:** local + CI.

## Step 8 — Config package + Zod environment validation
- **Create/modify:** `packages/config/src/{base.schema.ts,api.schema.ts,bot.schema.ts,worker.schema.ts,dashboard.schema.ts,load.ts,redact.ts}`; per-app `.env.example` (only variables that service needs — see wispbyte.md §4); `infrastructure/scripts/doctor.ts` skeleton.
- **Dependencies:** `zod`, `dotenv` (dev only; production reads real environment).
- **Commands:** `pnpm --filter @devland/config test` · `pnpm doctor --service api` (dummy env should fail loudly)
- **Expected:** valid env → typed config; invalid/missing → process exits with a readable list of problems and **no secret values printed**.
- **Fails if:** a service starts with missing secrets, defaults exist for secrets, `process.env` read elsewhere (lint catches).
- **Fix before continuing:** tighten schemas.
- **Tests required:** missing/invalid cases per service; `NODE_ENV`/`APP_ENV` mismatch refused; production rejects placeholder secrets and requires distinct keys; `DB_POOLER_MODE=transaction` without `DATABASE_LOCK_URL` refused; `STORAGE_DRIVER=local` in production refused unless `STORAGE_LOCAL_SHARED_VERIFIED=true`; `REDIS_NOEVICTION_VERIFIED` only honored when `CONFIG GET` is unavailable; `CRYPTO_KEYS` must decode to 32 bytes; redaction never prints values; ports come from env only (no hardcoded port in code).
- **Scope:** local + CI + every deploy target.

## Step 9 — Infrastructure interfaces (`packages/infra`)
- **Create/modify:** `src/{clock,ids,logger,secrets,cache,queue,lock,rate-limit,health,shutdown-types}.ts` (interfaces + capability flags only; drivers come in later steps); `packages/testing/src/contracts/*` reusable contract suites.
- **Dependencies:** `pino` (logger driver).
- **Commands:** `pnpm --filter @devland/infra build test`
- **Expected:** interfaces compile; in-memory fakes pass the contract suites.
- **Fails if:** interfaces leak vendor types (ioredis, BullMQ, S3, Discord) — they must not.
- **Fix before continuing:** keep vendor types inside drivers.
- **Tests required:** contract suites run against fakes; logger redaction test (fields named `token|secret|password|authorization|cookie` masked).
- **Scope:** local + CI.

## Step 10 — Crypto / keyring
- **Create/modify:** `packages/infra/src/crypto/{keyring.ts,envelope.ts,field.ts,hmac.ts,hash.ts,ip.ts}`; `infrastructure/scripts/rotate-keys.ts`. Format `enc:v1:<keyId>:<iv>:<tag>:<ciphertext>` (base64url), AES-256-GCM, random 96-bit IV, optional AAD (e.g. `table.column.rowId`); per-file data keys wrapped by keyring key; `hashIp()` = HMAC(pepper, ip); constant-time compare. **Node `crypto` only.**
- **Dependencies:** none external.
- **Commands:** `pnpm --filter @devland/infra test -t crypto`
- **Expected:** round-trip OK; tampering detected.
- **Fails if:** IV reuse, decrypt succeeds after bit flip, wrong AAD accepted, plaintext in logs/errors.
- **Fix before continuing:** fix primitives; do not invent algorithms.
- **Tests required:** known-answer vectors; flip any byte (iv/tag/ct) → fail; wrong key id → typed error; AAD mismatch → fail; rotation (new active key; old ciphertext still decrypts; `rotate-keys` re-encrypts and is idempotent); 10k encryptions → no IV collisions; ip hash stable per pepper, differs across peppers.
- **Scope:** local + CI. *Honest limit documented in the package README: rotation is operator-triggered; automatic rotation needs an external KMS.*

## Step 11 — Cache / queue abstractions (Redis drivers)
- **Create/modify:** `packages/infra/src/redis/{connection.ts,cache.ts,lock.ts,rate-limit.ts,queue.ts,health.ts}`; BullMQ queue/worker factories with default job options (attempts, exponential backoff + jitter, removeOnComplete limits, `jobId` support); fencing-token lock with safe release script; sliding-window limiter (Lua); bot leader lock helper; startup check for `maxmemory-policy noeviction` (warn/fail by config).
- **Dependencies:** `ioredis`, `bullmq`.
- **Commands:** `docker compose … up -d redis` · `pnpm --filter @devland/infra test -t redis`
- **Expected:** contract suites pass against real Redis.
- **Fails if:** lock released by a non-owner, limiter allows bursts over limit, vendor types leak.
- **Fix before continuing:** fix Lua/scripts and wrappers.
- **Tests required:** two contenders → exactly one holds the lock; expiry then takeover; non-owner release is a no-op; rate limit boundaries; duplicate `jobId` deduped; **Redis stopped mid-test → cache returns a miss / typed `CacheUnavailableError`, and DB writes (incl. outbox rows) still commit** (the "Redis failure must not lose business data" requirement); reconnect works.
- **Scope:** local + CI.

## Step 12 — Storage abstraction (`packages/storage`)
- **Create/modify:** `src/{provider.ts,local.driver.ts,s3.driver.ts,encrypted-store.ts,keys.ts,index.ts}` implementing the doc 01 §5.3 interface and capability flags; `EncryptedObjectStore` (envelope-encrypt → put; get → decrypt → verify SHA-256); key builder `{classification}/{yyyy}/{mm}/{uuid}`; **no `wispbyte` driver** (stub file with a TODO and a `NotImplementedError`, documented).
- **Dependencies:** `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner` (only used when the `presignedUrls` capability is true).
- **Commands:** `docker compose … up -d minio` · `pnpm --filter @devland/storage test`
- **Expected:** the same contract suite passes for `local` and `s3` (MinIO).
- **Fails if:** path traversal possible, partial writes visible, public ACL set, plaintext on disk/bucket.
- **Fix before continuing:** harden drivers.
- **Tests required:** put/get/head/delete; `../` and absolute keys rejected; atomic write (temp + rename; kill mid-write leaves no partial object); SHA mismatch detected; stored bytes ≠ plaintext; large stream with bounded memory; temp URL only when capability true; S3 puts set no public ACL; max size enforced; capability flags accurate.
- **Scope:** local + CI (and WispByte only if/when a driver is added).

## Step 13 — Audit system (`packages/audit`)
- **Create/modify:** `src/{canonical.ts,hash.ts,writer.ts,checkpoint.ts,verifier.ts,redact.ts}`; `infrastructure/scripts/verify-audit-chain.ts`. `AuditWriter.record(tx, event)` takes `pg_advisory_xact_lock` **on the connection obtained from `LockConnectionProvider` (never a transaction-pooled one)**, reads last `(sequence,hash)`, writes next row with `prevHash`/`hash`; checkpoint signer uses the separate `AUDIT_SIGNING_KEY` (**only the worker needs it**); verifier recomputes the chain and checkpoint signatures.
- **Dependencies:** none external.
- **Commands:** `pnpm --filter @devland/audit test` · `pnpm tsx infrastructure/scripts/verify-audit-chain.ts`
- **Expected:** chain verifies; tampering is located precisely.
- **Fails if:** gaps/forks under concurrency, canonicalization unstable, verifier misses tampering.
- **Fix before continuing:** fix locking/canonicalization.
- **Tests required:** golden canonical-JSON vectors (stable across versions); 50 concurrent writers → gapless linear chain; superuser edit/delete of a row → verifier reports first broken sequence; checkpoint with wrong key fails; triggers block UPDATE/DELETE for the app role; redaction removes secret/PII fields from `before/after`; rolled-back business tx leaves no audit row.
- **Scope:** local + CI + worker runtime. *Docs/UI must say "tamper-evident", not "immutable".*

## Step 14 — Event platform (`packages/events` + `packages/contracts`)
- **Create/modify:** `contracts/src/events/*/v1.ts` for the Phase 1 MVP catalog (schemas only), `events/src/{envelope.ts,registry.ts,context.ts,bus.ts}`; `AsyncLocalStorage` correlation/causation context; `EventBus.publish(tx, event)` validates against the registry and inserts an `OutboxEvent` row; scripts `events:docs` (generate catalog) and `events:compat` (compare with previous release).
- **Dependencies:** `zod`.
- **Commands:** `pnpm --filter @devland/events test` · `pnpm events:compat`
- **Expected:** registry complete; compat check passes against baseline.
- **Fails if:** an event is published without a registered schema, payload contains forbidden fields (secrets/proofs), compat script cannot detect a breaking change.
- **Fix before continuing:** fix registry and schemas.
- **Tests required:** invalid payload rejected; unknown type rejected; duplicate registration rejected; correlation id propagates across async boundaries and into the outbox row; compat checker fails on a fixture with a removed/renamed required field; decimals are strings, ids are uuids.
- **Scope:** local + CI.

## Step 15 — Outbox / Inbox / DLQ
- **Create/modify:** `events/src/{relay.ts,consumer.ts,dlq.ts,replay.ts,reconcile.ts}`, worker wiring in `apps/worker/src/queues/*` (queues from doc 04 §3, foundation ones only: `outbox-relay`, `dlq`, `timers` skeleton); `infrastructure/scripts/events-replay.ts`.
- **Dependencies:** `bullmq`, `ioredis` (via infra).
- **Commands:** `pnpm --filter @devland/events test` (Postgres + Redis up)
- **Expected:** at-least-once delivery, effectively-once processing, DLQ on exhaustion.
- **Fails if:** an event is lost when Redis is down, a consumer applies an effect twice, events reorder within a partition.
- **Fix before continuing:** fix transactions/idempotency.
- **Tests required:** crash between "added to queue" and "marked published" → no duplicate effect; same event delivered N times → one effect (`InboxEvent` insert and handler effect in one tx); handler throws → backoff retries → `DeadLetterEvent` after max attempts; **Redis down during publish → business tx commits, event delivered after Redis returns**; two relays at once → no double publish; per-partition ordering preserved; `availableAt` delay honored; replay is safe; DLQ retry/resolve works.
- **Scope:** local + CI + worker runtime.

## Step 16 — RBAC / ABAC (`packages/access`)
- **Create/modify:** `contracts/src/permissions.ts` (catalog from doc 03 §1.4), `access/src/{evaluator.ts,authorization.service.ts,role-mapping.repo.ts,grants.repo.ts,cache.ts,visibility.ts}`, seed script for `Permission` and system roles, Nest guard + `@RequirePermission` in `apps/api` (kept out of the framework-agnostic package).
- **Dependencies:** none external (the condition evaluator is a hand-written pure interpreter; **no `eval`/`Function`**).
- **Commands:** `pnpm --filter @devland/access test`
- **Expected:** the decision matrix passes.
- **Fails if:** default is not deny, a DENY can be overridden by an ALLOW, expired grants still count, malformed conditions crash or allow.
- **Fix before continuing:** fix evaluator order.
- **Tests required:** table-driven matrix over the catalog × role profiles (e.g. developer can never `payment.proof.view`; customer reads only own orders); deny overrides allow; role inheritance; expired/revoked grant ignored; sensitive permission without step-up → `STEP_UP_REQUIRED`; lockdown denies non-allowlisted actions; ABAC cases (pool match, amount threshold, ownership); malformed/oversized condition JSON denied safely; cache invalidation on mapping change; decision trace emitted; role mapping never calls any Discord role-creation API.
- **Scope:** local + CI.

## Step 17 — Authentication, sessions, TOTP (`packages/auth`)
- **Create/modify:** `auth/src/{discord-oauth.ts,session.service.ts,totp.service.ts,recovery.service.ts,csrf.ts,step-up.ts,owner-bootstrap.ts}`, `apps/api/src/http/auth.controller.ts`; `SecurityEvent` writes; IP hashing via step 10.
- **Dependencies:** `otplib`, `argon2` (or SHA-256 for high-entropy recovery codes — document the choice), `undici`.
- **Commands:** `pnpm --filter @devland/auth test`
- **Expected:** full login → 2FA → session → step-up → logout flow works against a **fake Discord OAuth server** (no real Discord calls in tests).
- **Fails if:** raw session token stored, reusable TOTP code accepted, OAuth state not checked, session not rotated after 2FA.
- **Fix before continuing:** fix flow.
- **Tests required:** only `SHA-256(token)` in DB; idle and absolute expiry; revoke one / revoke all; TOTP window accepted, replay of same step rejected; recovery code single-use; wrong OAuth `state`/PKCE rejected; user without `dashboard.access` rejected + security event; owner bootstrap only for `OWNER_DISCORD_ID`; new session id after 2FA/step-up; CSRF rejects missing/foreign origin; login failures rate-limited; no secrets in logs.
- **Scope:** local + CI + API runtime.

## Step 18 — Discord bot foundation (`apps/bot`)
- **Create/modify:** `src/{main.ts,gateway/client.ts,gateway/leader.ts,actor-context.ts,errors.ts,register-commands.ts}`, `DiscordGateway` **port** in `packages/infra` (so tests use a fake guild); structured logs; error boundary producing `errorId`; guild-scoped command registration as a **one-off job**, never at startup.
- **Dependencies:** `discord.js`.
- **Commands:** `pnpm --filter @devland/bot build test` · (manual, dev guild) `pnpm --filter @devland/bot start:dev`
- **Expected:** bot connects with the dev application, holds the leader lock, answers a ping interaction.
- **Fails if:** privileged intents requested by default (`GuildMembers` and `MessageContent` are privileged — keep them **off** unless config enables them; role data arrives with interactions), two instances both active, token logged.
- **Fix before continuing:** fix intents/leader lock.
- **Tests required:** with a fake gateway; second instance does not become active while the lock is held; takeover after lock expiry; intents list asserted (no privileged by default); `ActorContext` built from interaction member roles only; error boundary hides internals.
- **Scope:** local + CI + WispByte runtime.

## Step 19 — i18n (`packages/i18n`)
- **Create/modify:** `src/{ar.json,en.json,t.ts,locale.ts,plural.ts,keys.d.ts}`; locale resolution (stored `User.locale` → Discord interaction locale → default); Arabic plural rules via `Intl.PluralRules`; RTL helpers; CI script `i18n:check`.
- **Dependencies:** none (Node/Intl built-ins).
- **Commands:** `pnpm --filter @devland/i18n test` · `pnpm i18n:check`
- **Expected:** key parity AR/EN, typed keys, all keys used in code exist.
- **Fails if:** a key exists in one locale only, interpolation variable missing, hard-coded user-facing strings in renderers.
- **Fix before continuing:** add keys/translations (no machine-guessed Arabic shipped without owner review: mark `needsReview`).
- **Tests required:** parity; plurals for 0/1/2/3–10/11+; fallback to default; missing variable throws in tests; ICU-like interpolation escapes Discord markdown/mentions in user-provided values (prevents mention injection).
- **Scope:** local + CI + Vercel (dashboard also uses it).

## Step 20 — Interaction router
- **Create/modify:** `apps/bot/src/interactions/{custom-id.ts,router.ts,middleware/*,registry.ts}`; custom id format `dl:<domain>:<action>:v<N>:<payload>` ≤ 100 chars; middleware chain: correlation id → defer within budget → idempotency (`interaction.id`) → rate limit → resolve actor → authorize → handler → error boundary.
- **Dependencies:** none beyond step 11/16.
- **Commands:** `pnpm --filter @devland/bot test`
- **Expected:** router dispatches commands/buttons/selects/modals through the same pipeline.
- **Fails if:** payload trusted without re-authorizing, id > 100 chars, double click double-executes.
- **Fix before continuing:** fix middleware order.
- **Tests required:** custom-id round trip; over-length rejected at build time; unknown/old version → safe ephemeral message; tampered payload cannot escalate (server re-checks permission/ownership); same interaction id twice → single execution; authz failure leaks nothing; ack/defer within 3 s (fake timers); errors return `errorId` only.
- **Scope:** local + CI + WispByte runtime.

## Step 21 — `/setup` skeleton
- **Create/modify:** `packages/core/src/modules/discord-setup/{setup.plan.ts,setup.service.ts,layout.ts}` using the `DiscordGateway` port; `apps/bot/src/interactions/commands/setup.ts`; layout data file (categories/channels/placeholder panels; no business panels yet).
- **Dependencies:** none new.
- **Commands:** `pnpm --filter @devland/core test -t setup` · manual run in dev guild: `/setup dry-run`, then `/setup apply`, then `/setup apply` again.
- **Expected:** `plan()` shows a diff; `apply()` is idempotent; second apply changes nothing.
- **Fails if:** duplicates created, any role-creation call made, setup runs without `setup.run`.
- **Fix before continuing:** fix idempotency/permissions.
- **Tests required:** with a fake guild — dry run mutates nothing; apply creates N items and records `DiscordChannelBinding`/`DiscordPanel`; rerun creates 0; deleted channel is re-created or flagged per policy; mapped roles used for overwrites, **unmapped role → owner-only overwrite + "map roles in dashboard" notice, never role creation (assert the fake's `createRole` is never called)**; partial failure resumes; audit event + `setup.completed` emitted; denied for users without `setup.run`.
- **Scope:** local + CI + WispByte runtime.

## Step 22 — Dashboard authentication shell
- **Create/modify:** `apps/dashboard/src/app/{login,auth/callback,2fa,(app)/layout,(app)/sessions}`, middleware (cookie presence gate only; real checks server-side), BFF proxy (D2: rewrites/route handlers to `API_BASE_URL`), CSRF token handling, error/not-found pages, `next.config.ts` security headers + CSP with nonces.
- **Dependencies:** `next`, `react`, `react-dom`, `@tanstack/react-query`, `zod`, `next-intl`; Playwright (dev).
- **Commands:** `pnpm --filter @devland/dashboard build test` · `pnpm --filter @devland/dashboard test:e2e` (API in test mode with fake OAuth)
- **Expected:** unauthenticated → login; owner login → forced 2FA enrolment → shell; sessions page lists/revokes; logout invalidates.
- **Fails if:** a secret reaches the client bundle, cookie not host-only/HttpOnly/Secure, CSP missing, proxy drops `Set-Cookie`.
- **Fix before continuing:** fix proxy/headers; confirm D2 works on a Vercel preview.
- **Tests required:** Playwright flows above; bundle scan for forbidden strings (`DISCORD_BOT_TOKEN`, `CRYPTO_KEYS`, `DATABASE_URL`); header assertions; CSRF negative test; RTL snapshot.
- **Scope:** CI + Vercel.

## Step 23 — Dashboard "DevLand Premium" design foundation
- **Create/modify:** `packages/ui/src/{tokens.css,tailwind-preset.ts,components/*,motion/*}`; components: Button, Card (glass), Input, Select, Table (responsive), Sidebar, Topbar, Modal, Toast, Skeleton, StatCard, Chart wrapper; GSAP presets honoring `prefers-reduced-motion`; logical properties for RTL; web fonts via `next/font` (self-hosted, Arabic-capable); PWA manifest + service worker (offline shell only; **never cache authenticated API responses**).
- **Dependencies:** `tailwindcss`, `gsap`, `@gsap/react`, `lucide-react`, `recharts`, `@serwist/next`; dev: `@testing-library/react`, `axe-core`/`jest-axe`.
- **Commands:** `pnpm --filter @devland/ui test` · `pnpm --filter @devland/dashboard build` · open the dev server for visual review.
- **Expected:** consistent dark futuristic system (glass, subtle depth, premium typography), mobile-first, AR/EN RTL/LTR working, installable PWA shell.
- **Fails if:** it reads as a generic admin template (owner's explicit requirement — design review checkpoint), contrast failures, motion ignores reduced-motion, service worker caches private data.
- **Fix before continuing:** iterate with the owner; record screenshots in the PR.
- **Tests required:** component unit tests; axe accessibility checks; reduced-motion test; service-worker test proving authenticated endpoints are `network-only`; Playwright screenshots (informational, non-blocking).
- **Scope:** local + CI + Vercel.

## Step 24 — Health checks
- **Create/modify:** `apps/api/src/health/*`; `apps/bot/src/health/server.ts` and `apps/worker/src/health/server.ts` (tiny `node:http` servers); dashboard `/api/health`. Endpoints: `/health/live` (process up; no dependencies), `/health/ready` (DB, Redis, storage with per-check timeouts; 503 + failing component name, never connection strings), `/health/detail` (needs `HEALTH_TOKEN`). **Ports come from env** (`PORT` for api; `HEALTH_PORT` for bot/worker); nothing hardcoded.
- **Dependencies:** `@nestjs/terminus` (api) or plain HTTP.
- **Commands:** `pnpm --filter @devland/api test -t health` and equivalents; `curl` the endpoints against compose services.
- **Expected:** live=200 always while the process runs; ready=200 only when dependencies are healthy.
- **Fails if:** `/live` depends on DB (restart loops), details leak secrets, checks hang without timeout.
- **Fix before continuing:** fix indicators/timeouts.
- **Tests required:** each indicator OK/fail/timeout; DB stopped → ready 503, live 200; Redis stopped → ready degraded per config; `/detail` rejects missing/wrong token; response bodies contain no secrets.
- **Scope:** local + CI + WispByte (platform health-check config unverified) + Vercel (dashboard).

## Step 25 — Graceful shutdown
- **Create/modify:** `packages/infra/src/shutdown.ts` (`ShutdownManager`: ordered hooks, global timeout, second-signal force exit, exit codes); wiring in all three apps: **api** stop accepting → drain in-flight → close Prisma/Redis; **bot** stop accepting interactions → finish in-flight → destroy client → release leader lock; **worker** pause queues → wait for active jobs (bounded) → release locks → close connections.
- **Dependencies:** none.
- **Commands:** `pnpm --filter @devland/worker test -t shutdown` (spawns the built process and sends signals).
- **Expected:** SIGTERM → in-flight work completes → exit 0 within timeout.
- **Fails if:** jobs killed mid-flight with no recovery, locks left held, process hangs past timeout.
- **Fix before continuing:** fix hook ordering.
- **Tests required:** SIGTERM during an active job → job completes, exit 0; second SIGTERM → forced exit non-zero; SIGKILL during a job → after restart the reconciler/redelivery completes it exactly once; leader lock released on clean exit; hooks run in order even if one throws.
- **Scope:** local + CI + WispByte (stop-signal behavior on the host is unverified — see wispbyte.md §1).

## Step 26 — CI validation (`.github/workflows`)
- **Create/modify:** `ci.yml` (install with pnpm cache → typecheck → lint → depcruise → `prisma validate` + format check → migrate on fresh Postgres + drift check → unit tests → integration tests with Postgres/Redis/MinIO service containers → `i18n:check` → `events:compat` → build all), `security.yml` (gitleaks, `pnpm audit`, dependency review), `release.yml`, `CODEOWNERS` (db, access, auth, audit, payments, events, infrastructure), Renovate/Dependabot config; branch protection with required checks.
- **Dependencies:** GitHub Actions, gitleaks.
- **Commands:** open a PR; additionally a throwaway PR with a deliberate failure (fake key in a file, type error, depcruise violation).
- **Expected:** green on the real PR; each deliberate failure turns CI red.
- **Fails if:** CI green despite an injected failure, flaky integration tests, secrets visible in logs.
- **Fix before continuing:** fix workflows; tests must be deterministic.
- **Tests required:** the injected-failure PR is the test.
- **Scope:** CI.

## Step 27 — Production build validation
- **Create/modify:** `infrastructure/scripts/verify-prod-build.sh`; per-app `build`/`start` scripts (`node dist/main.js`); `engines` fields.
- **Dependencies:** none new.
- **Commands:** `NODE_ENV=production pnpm turbo run build` · for each of api/bot/worker: `pnpm --filter @devland/<app> deploy --prod artifacts/<app>` then, **from inside `artifacts/<app>` with no repo access**, run `node dist/main.js` against compose services with a prod-shaped env · `curl /health/ready` · dashboard: `next build` with prod env.
- **Expected:** each service boots from its artifact with prod dependencies only and reports ready. **Record `pnpm -v` and the exact `pnpm deploy` behavior; artifacts remain PROVISIONAL (not for production) until D5 is verified AND the Prisma host target is known (rule 5).**
- **Fails if:** a devDependency is needed at runtime, Prisma client/engine missing for the target platform, TypeScript sources/tests present in `dist`, `.env` files inside the artifact.
- **Fix before continuing:** fix dependency classification and the generate/packaging step (D3).
- **Tests required:** boot-from-artifact smoke test per service; script asserts no `*.test.*`, `*.map` (policy), `.env*`, or `devDependencies` in the artifact; report artifact sizes.
- **Scope:** local + CI (+ proves WispByte/Vercel readiness).

## Step 28 — Deployment artifact generation
- **Create/modify:** `infrastructure/scripts/build-artifact.sh <api|bot|worker>` producing `dist-artifacts/devland-<svc>-<version>-<sha>.tgz` + `artifact-manifest.json` (service, version, git SHA, Node version, build time, file SHA-256s); per-service multi-stage `Dockerfile` (**used only if the host runs container images — unverified**); `.dockerignore`; `infrastructure/deployment/templates/*.env.template` (names only); `release.yml` uploads artifacts as workflow artifacts (not public).
- **Dependencies:** none new.
- **Commands:** `infrastructure/scripts/build-artifact.sh api` (then bot, worker) · extract into a clean temp dir and boot (as step 27) · `gitleaks detect --no-git --source <extracted dir>`
- **Expected:** three self-contained artifacts with manifests, secret-free, bootable on their own. Manifests carry `"provisional": true` while D4/D5 or the Prisma host target are unverified; `release.yml` must refuse to publish provisional artifacts to a production environment.
- **Fails if:** script finds `.env*`/secrets, an artifact boots only with repo files present, manifest hashes mismatch.
- **Fix before continuing:** fix exclusion list and packaging.
- **Tests required:** extract-and-boot smoke test; secret scan on each artifact; manifest verification; size report in `phase-2-report.md`.
- **Scope:** CI + local; deploys to WispByte only after its capabilities are verified.

---

## Phase 2 exit criteria
All 28 steps recorded in `docs/phase-2-report.md` with real command output; CI green on `main`; Prisma validate/format/migration/drift proven; every test listed above exists and passes; production artifacts boot standalone; unresolved infrastructure assumptions listed. Then **stop and report** — do not begin Phase 3 (store, orders, tickets, payments business logic) without approval.

## `docs/phase-2-report.md` template
```md
| Step | Command | Exit | Result (trimmed output) | Date | SHA | Notes/Issues |
|---|---|---|---|---|---|---|
```
Plus sections: Prisma validation result · Migration result · Tests run (counts) · Typecheck result · Lint result · Build result · Artifact sizes · Unresolved issues · Exact next phase.
