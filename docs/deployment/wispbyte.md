# Deploying DevLand services on WispByte

**Status of this document: planning template. No WispByte capability below is verified.** Everything written as a *requirement* is what DevLand needs. Whether WispByte provides it is **UNVERIFIED** until you confirm it in WispByte's panel/documentation or with a test deployment and record the answer in §1. The code never hardcodes WispByte URLs, ports, paths or credentials; all of that arrives through environment variables and adapters, so the same artifacts run locally, on WispByte or on another host.

Targets: `api`, `bot`, `worker` → WispByte. `dashboard` → Vercel (see `vercel.md`). PostgreSQL, Redis and object storage → provider adapters (host decided after §1).

---

## 1. Capability verification (do this before deploying anything)

Fill the "Result" column from real evidence (docs link, panel screenshot, test run). Do not guess.

| # | Capability DevLand needs | Why it matters | How to verify | If NOT available | Result |
|---|---|---|---|---|---|
| 1 | Node runtime at the pinned LTS version (D4) | Artifacts are built for one Node version | Panel/docs; run `node -v` in a test deploy | Pick another host for that service or change D4 | ☐ |
| 2 | Run **3 separate long-running processes** (api, bot, worker) | Independent deployability | Create 3 services/apps or confirm multi-process support | Run on another VPS/PaaS; or combine only as a documented fallback (loses isolation) | ☐ |
| 3 | Start command configurable (`node dist/main.js`) | Production start | Panel setting | Wrapper script | ☐ |
| 4 | Upload/deploy method for a built folder/archive (or Git deploy) | We ship prod-only artifacts | Try uploading `devland-api-*.tgz` | CI → SFTP/rsync, or another host | ☐ |
| 5 | Docker/container image support | Only needed if we use the Dockerfiles | Docs | Use artifact deploy instead (default path) | ☐ |
| 6 | Inbound HTTPS for the API (custom domain `api.<domain>`, TLS) | Dashboard → API | Test endpoint | Put a proxy/CDN in front, or host API elsewhere | ☐ |
| 7 | How the listening port is assigned (env `PORT`? fixed?) | Port must come from env | Panel/docs | Set `PORT` explicitly if configurable | ☐ |
| 8 | **Outbound** connections to Postgres, Redis, Discord (gateway websocket + REST), S3 | Core functionality | Test deploy opening TCP/TLS to each | Host the failing service elsewhere | ☐ |
| 9 | Secret/environment variable storage (not in files in the repo) | Secrets stay out of Git | Panel | Secret injection from CI at deploy time | ☐ |
| 10 | Persistent disk (needed only for the `local` storage driver fallback) | Local storage fallback | Write file, restart, check it survives | Use S3-compatible storage instead | ☐ |
| 11 | Auto-restart on crash + restart policy controls | Self-recovery | Kill process, observe | External supervisor / health-check-based restarts | ☐ |
| 12 | Stop signal (SIGTERM?) and grace period before kill | Graceful shutdown (runbook step 25) | Deploy a test app that logs signals | Shorten shutdown timeouts; rely on reconcilers | ☐ |
| 13 | Log access (stdout/stderr capture, retention, download) | Operations | Panel | Ship logs to an external sink | ☐ |
| 14 | Memory/CPU limits | Sizing (Prisma + BullMQ + discord.js) | Panel | Reduce concurrency settings | ☐ |
| 15 | Scheduled / one-off jobs (cron, run command) | Migrations, backups, verifications | Panel | Run from CI or a local machine over a secure connection | ☐ |
| 16 | Does it offer managed PostgreSQL / Redis / storage? Versions, TLS, backups, connection limits, `noeviction`/persistence for Redis | Adapter targets | Docs/panel | Use external providers (e.g. any managed Postgres/Redis/S3-compatible) | ☐ |
| 17 | Multiple instances / zero-downtime deploy / scaling | Zero-downtime plan (§12) | Docs | Accept brief restarts; reconcilers cover gaps | ☐ |
| 18 | Platform health-check probe support | Auto-restart on failure | Panel | Use external uptime monitor | ☐ |

**Decision rule:** the service layout in §2 is the target. For any row marked unavailable, apply the fallback column and record it in an ADR. Never write WispByte-specific code until rows 1–4, 7–9 and 11–12 are confirmed.

---

## 2. Recommended service layout

| Service | Process | Instances | Restart | Needs inbound HTTP |
|---|---|---|---|---|
| `devland-api` | `node dist/main.js` (NestJS/Fastify) | 1 (more if platform supports it; stateless) | always | **Yes** (HTTPS to `api.<domain>`) |
| `devland-bot` | `node dist/main.js` (discord.js) | **exactly 1 active** (leader lock enforces) | always | Only the internal health port, if the platform needs a probe |
| `devland-worker` | `node dist/main.js` (BullMQ consumers + schedulers + outbox relay) | 1+ (queues can be split via `WORKER_QUEUES`) | always | Only the internal health port, if needed |
| PostgreSQL | external or provided (§8) | — | — | — |
| Redis | external or provided (§9) | — | — | — |
| Object storage | S3-compatible or `local` fallback (§10) | — | — | — |

---

## 3. What to upload — and what NOT to

Build artifacts in CI or locally (`runbook` steps 27–28). Upload **only** the artifact for that service.

| Upload to WispByte | Contents |
|---|---|
| `devland-api-<ver>-<sha>.tgz` | `dist/`, production `node_modules` (incl. generated Prisma client/engine for the host platform), `package.json`, `artifact-manifest.json` |
| `devland-bot-<ver>-<sha>.tgz` | same shape |
| `devland-worker-<ver>-<sha>.tgz` | same shape |

**Never upload:** the monorepo, `docs/`, `**/*.test.*`, `packages/testing`, `packages/tooling`, `packages/ui` and `apps/dashboard` (Vercel only), `infrastructure/monitoring`, `.github`, `.git`, `.env*`, `pnpm-lock.yaml`/workspace files (the artifact is already self-contained), TypeScript sources, local Docker compose files. Secrets are set in the platform's environment settings, **never inside an artifact**.

Prisma note: the generated client contains a native query engine. `binaryTargets` in the schema must include the host's OS/OpenSSL target (**unknown until §1 row 1/16 is answered**); wrong target = service fails at first query.

---

## 4. Environment variables per service

Names only; real values live in the host's secret store. Variables marked 🔒 are secrets. Each service validates its own set at boot (zod) and refuses to start if invalid. Least privilege: a service receives only what it needs.

| Variable | api | bot | worker | Notes |
|---|:-:|:-:|:-:|---|
| `NODE_ENV`, `APP_ENV`, `LOG_LEVEL`, `SERVICE_NAME` | ✔ | ✔ | ✔ | `APP_ENV` must match the environment |
| `DEFAULT_LOCALE`, `OWNER_DISCORD_ID` | ✔ | ✔ | — | |
| `PORT` | ✔ | — | — | from platform or set explicitly |
| `HEALTH_PORT`, `HEALTH_TOKEN` 🔒 | detail only | ✔ | ✔ | health server for bot/worker |
| `DATABASE_URL` 🔒 | ✔ | ✔ | ✔ | pooled URL |
| `DB_SUPPORTS_ADVISORY_LOCKS`, `DB_SUPPORTS_ROLE_REVOKE` | ✔ | ✔ | ✔ | capability flags |
| `DB_POOLER_MODE` (`none`\|`session`\|`transaction`), `DATABASE_LOCK_URL` 🔒 | ✔ | ✔ | ✔ | transaction mode without a lock URL = refuse to boot |
| `STORAGE_LOCAL_SHARED_VERIFIED` | if `local` | — | if `local` | must stay `false`/unset unless shared persistent storage is verified |
| `REDIS_NOEVICTION_VERIFIED` | optional | optional | optional | only if `CONFIG GET` is blocked and the provider setting was checked |
| `REDIS_URL` 🔒, `REDIS_KEY_PREFIX` | ✔ | ✔ | ✔ | `rediss://` if TLS |
| `STORAGE_DRIVER`, `STORAGE_BUCKET`, `STORAGE_MAX_UPLOAD_MB`, `STORAGE_TEMP_URL_TTL_SEC` | ✔ | — | ✔ | |
| `STORAGE_LOCAL_PATH` | if `local` | — | if `local` | persistent disk required |
| `S3_ENDPOINT`, `S3_REGION`, `S3_ACCESS_KEY_ID` 🔒, `S3_SECRET_ACCESS_KEY` 🔒 | if `s3` | — | if `s3` | |
| `CRYPTO_KEYS` 🔒, `CRYPTO_ACTIVE_KEY_ID`, `IP_HASH_PEPPER` 🔒 | ✔ | ✔ | ✔ | same keyring across services that read the same data |
| `AUDIT_SIGNING_KEY` 🔒 | — | — | ✔ | only the checkpoint signer/verifier |
| `DISCORD_APPLICATION_ID`, `DISCORD_GUILD_ID` | ✔ | ✔ | — | |
| `DISCORD_BOT_TOKEN` 🔒 | — | ✔ | — | worker never needs it: Discord writes go through the `discord-outbound` queue consumed by the bot |
| `DISCORD_CLIENT_SECRET` 🔒, `DISCORD_OAUTH_REDIRECT_URI` | ✔ | — | — | dashboard login |
| `SESSION_*`, `STEP_UP_WINDOW_MINUTES`, `TOTP_ISSUER`, `CSRF_SECRET` 🔒 | ✔ | — | — | |
| `DASHBOARD_ORIGIN`, `API_BASE_URL` | ✔ | — | — | CORS/redirects |
| `OUTBOX_*`, `EVENT_MAX_ATTEMPTS`, `WORKER_QUEUES` | — | — | ✔ | |
| `BOT_LEADER_LOCK_TTL_SEC` | — | ✔ | — | |
| `RATE_LIMIT_*` | ✔ | ✔ | — | |
| `SENTRY_DSN`, `OTEL_*` | optional | optional | optional | |

---

## 5. Build commands (CI or local; not on WispByte)

```bash
pnpm install --frozen-lockfile
NODE_ENV=production pnpm turbo run build
infrastructure/scripts/build-artifact.sh api     # then bot, worker
```
Production hosts never run `pnpm install` for dev dependencies and never compile TypeScript. If WispByte insists on installing from `package.json` itself, the artifact's `package.json` contains only production dependencies and `npm ci --omit=dev`/`pnpm install --prod` must work offline-from-lockfile (**verify**; otherwise ship `node_modules` inside the artifact).

## 6. Production start commands

| Service | Command (run from the artifact root) | Port source |
|---|---|---|
| api | `node dist/main.js` | `PORT` |
| bot | `node dist/main.js` | `HEALTH_PORT` (optional) |
| worker | `node dist/main.js` | `HEALTH_PORT` (optional) |

Set `NODE_ENV=production`. Add `--enable-source-maps` only if you ship maps. No process manager inside the artifact: the platform (or Docker/systemd elsewhere) supervises.

## 7. Health-check endpoints and ports

| Service | Endpoint | Meaning | Use for |
|---|---|---|---|
| api | `GET /health/live` | process alive (no dependency checks) | restart decision |
| api | `GET /health/ready` | DB, Redis, storage OK within timeouts → 200/503 | traffic routing / uptime monitor |
| api | `GET /health/detail` + `Authorization: Bearer $HEALTH_TOKEN` | component details | operators only |
| bot, worker | same paths on `HEALTH_PORT` | live/ready (bot ready also means gateway connected + leader state) | platform probe if supported |

Ports: **no ports are hardcoded anywhere.** API listens on `PORT`; bot/worker health servers on `HEALTH_PORT` (disabled if unset). Whether WispByte assigns or exposes ports is UNVERIFIED (§1 row 7). Never expose bot/worker health ports publicly; if the platform cannot restrict them, leave `HEALTH_PORT` unset and rely on platform process supervision plus the bot's own heartbeat alerts.

## 8. PostgreSQL connection requirements
- Postgres ≥ 15 (matches CI image; confirm host version).
- TLS required for any non-local host (`sslmode=require` or stricter); verify certificate if the provider supplies a CA.
- Two URLs: `DATABASE_URL` (pooled, used by apps) and `DIRECT_DATABASE_URL` (direct, used **only** by migration jobs, never set on running services).
- **Advisory-lock requirement (hard rule):** the audit writer and outbox-relay leader election use PostgreSQL advisory locks, which are session-scoped and **unsafe through transaction-mode pooling**. Set `DB_POOLER_MODE` truthfully; locks run on a dedicated connection from `DATABASE_LOCK_URL` (direct or session-mode). With `DB_POOLER_MODE=transaction` and no lock URL the service refuses to boot. `doctor` proves lock contention on the real path.
- **Connection budget:** (api pool + bot pool + worker pool) × instances must stay below the provider's connection limit; defaults: api 10, bot 5, worker 10 (set via URL params/config). Use a pooler (e.g. PgBouncer in transaction mode) only if it supports the advisory-lock pattern used by the audit writer — **advisory locks do not work reliably through transaction-mode pooling**: use the direct URL for the audit writer path or session-mode pooling (flag `DB_SUPPORTS_ADVISORY_LOCKS` must be true and tested by `doctor`).
- Separate DB roles: migration role (DDL) vs application role (DML, no UPDATE/DELETE on append-only tables when `DB_SUPPORTS_ROLE_REVOKE=true`).
- Network: only the three services (and your migration runner) may connect; no public access with a password-only login if the provider allows IP allowlisting.

## 9. Redis connection requirements
- `REDIS_URL` with TLS (`rediss://`) for remote hosts.
- **`maxmemory-policy noeviction`** (BullMQ requirement) must be **verified, not assumed**: boot/`doctor` read `CONFIG GET maxmemory-policy`; production refuses to start consumers otherwise (explicit `REDIS_NOEVICTION_VERIFIED=true` only when the provider blocks `CONFIG`). If the provider cannot set it, `doctor` fails in production.
- Persistence (AOF/RDB) recommended to reduce re-enqueue work after restarts, but **not relied upon**: Postgres outbox + reconcilers rebuild jobs/timers, so a full Redis loss is a latency event, not data loss.
- Never share one Redis between prod and non-prod; if unavoidable, `REDIS_KEY_PREFIX` must differ and ACL users should be separate.
- Memory sizing: queue payloads are small (ids), so modest sizes suffice; monitor memory.

## 10. Storage configuration
- Preferred: `STORAGE_DRIVER=s3` against an S3-compatible provider with **private** buckets, server-side encryption if offered (we also encrypt client-side), versioning/object-lock optional (only then WORM can be claimed).
- Fallback: `STORAGE_DRIVER=local` (**refused in production unless `STORAGE_LOCAL_SHARED_VERIFIED=true`**; never assume api and worker share a disk) with `STORAGE_LOCAL_PATH` on a **persistent** disk. Requirements: survives restarts and redeploys (verify §1 row 10), not inside the artifact directory (redeploy would wipe it), backed up (§14), single writer set (api + worker must see the *same* disk, otherwise local storage is invalid across processes — if they run on separate hosts, local is **not** usable).
- A `wispbyte` driver does not exist and will only be written after the platform's storage API is confirmed.
- Files are never public; delivery happens through permission-checked API streaming or short-lived signed URLs when the provider supports them.

## 11. Worker requirements
- At least one worker running at all times (outbox relay, timers, notifications, reconcilers). The outbox relay uses a leader/advisory lock, so extra workers are safe.
- Split heavy queues onto separate processes via `WORKER_QUEUES` when needed (e.g. PDF generation later).
- Clock accuracy: use NTP-synced hosts; timers are DB-reconstructed so drift causes lateness, not loss.
- Memory: bounded concurrency per queue (config); no unbounded in-memory buffers.

## 12. Restart policy, graceful shutdown, zero-downtime

**Restart policy:** always restart on crash with backoff; alert if >3 restarts in 10 minutes (monitoring §15). A crash loop must not hammer Postgres: boot-time config validation fails fast and exits non-zero (the platform's backoff applies).

**Graceful shutdown (implemented in runbook step 25):** on SIGTERM — api stops accepting and drains in-flight requests; bot stops taking interactions, finishes in-flight, releases the leader lock, destroys the client; worker pauses queues, waits for active jobs (bounded by `SHUTDOWN_TIMEOUT_MS`), releases locks. Set the shutdown timeout **below the platform's kill grace period** (UNVERIFIED, §1 row 12). A forced kill is survivable: jobs redeliver and are idempotent.

**Zero-downtime considerations (honest version):**
- Schema changes follow **expand → migrate → contract** across releases, so the previous app version keeps working on the new schema (this is what makes rollback safe).
- **api:** truly zero-downtime only if the platform can run two instances behind one endpoint (UNVERIFIED). Otherwise expect a few seconds of restart; Discord interactions are unaffected because the bot talks to Postgres directly, and dashboard requests retry.
- **bot:** a singleton. Rolling deploy = start new → it waits for the leader lock → stop old (releases lock) → new takes over. Discord interactions during the gap (typically seconds) may fail with "interaction failed"; accepted and documented.
- **worker:** drain old, start new; unfinished work is redelivered or reconciled.
- Order of rollout: migrations (expand only) → worker → api → bot → dashboard (Vercel).
- Maintenance windows/lockdown (later phases) can freeze sensitive operations before risky deploys.

## 13. What runs continuously vs one-off

| Continuous | One-off jobs (run from CI, a local machine, or a platform job runner if available) |
|---|---|
| `api`, `bot`, `worker` | `prisma migrate deploy` (using `DIRECT_DATABASE_URL`) |
| managed Postgres, Redis | Discord command registration (`register-commands`) |
| uptime monitor / alerting | seed (permissions, system roles, default settings) — idempotent |
| | `rotate-keys`, `verify-audit-chain`, `events-replay` |
| | database backup + restore test |
| | `doctor --service <name>` pre-flight check |

## 14. Backups
- **PostgreSQL:** automated daily logical or physical backups (provider-managed if available — verify retention and restore procedure; otherwise a scheduled one-off job running `pg_dump` over TLS, output **encrypted** with a key kept outside the backup location). Additionally a snapshot immediately before every production migration. Weekly/monthly retention per your spec; keep at least one copy off the primary provider.
- **Object storage:** provider versioning/replication if available; otherwise periodic sync to a second location (encrypted).
- **Redis:** not a backup target (reconstructable).
- **Secrets/keyring:** `CRYPTO_KEYS` and `AUDIT_SIGNING_KEY` must be backed up **separately and securely**; losing them makes encrypted data and audit verification unrecoverable.
- **Restore test:** quarterly (and before go-live) restore into an isolated database, run `verify-audit-chain` and a smoke test. A backup that was never restored is not a backup.

## 15. Logs and monitoring
- JSON logs to **stdout/stderr** with `timestamp, service, level, correlationId, requestId, userId?, orderId?, ticketId?, errorId?`. No secrets/PII (redaction enforced). Whether WispByte retains/exports them is UNVERIFIED (§1 row 13); if retention is short, forward to an external log sink.
- Alerts (minimum): `/health/ready` failing >2 min; restart storm; DLQ non-empty; outbox backlog age > threshold; queue depth growth; audit integrity violation; failed backup; Postgres/Redis connection errors; disk nearly full (if `local` storage). Delivery: Discord alert channel + one external channel (email/uptime service) so an outage of the bot itself is still noticed.
- External uptime monitor on `api.<domain>/health/ready` and the dashboard URL.

## 16. Migration procedure
1. Merge code to `main`; CI builds artifacts; migrations were already proven on a fresh DB with a drift check.
2. Announce window if the migration is non-trivial; take a **fresh backup/snapshot**.
3. Run `doctor` against production config (read-only checks).
4. Run `prisma migrate deploy` as a one-off job with `DIRECT_DATABASE_URL`, from CI or a trusted machine. Migrations must be **expand-only** (add tables/columns/indexes nullable or defaulted). Create large indexes `CONCURRENTLY` in a dedicated migration.
5. Deploy worker → api → bot (artifacts), verifying `/health/ready` after each.
6. Run smoke checks (login, a read-only API call, bot ping).
7. Later release: contract step (drop/rename) once no running version uses the old shape.

## 17. Rollback procedure
- **Application:** redeploy the previous artifact (keep the last 3 per service; manifests record version + SHA). Order: bot/api/worker in reverse rollout order. Safe because the schema is expand-only.
- **Database:** **no down-migrations in production.** Fix forward with a new migration, or — for data corruption — restore from the pre-migration snapshot into a new database, verify, then repoint `DATABASE_URL` (accepting loss of writes after the snapshot; reconcile from the audit/outbox logs).
- **Config/secrets:** keep previous values until the release is confirmed healthy.
- **Dashboard:** Vercel instant rollback (see `vercel.md`), after confirming API compatibility.
- Post-rollback: write an incident note; verify audit chain and outbox backlog.

---

## 18. Deployment checklist

Tick only with evidence (screenshot/log/command output).

**Pre-flight**
- [ ] §1 capability table completed; fallbacks recorded in ADRs
- [ ] Discord token from chat reset; separate prod application created
- [ ] Node version on host = pinned LTS
- [ ] `binaryTargets` match the host platform

**API**
- [ ] Artifact built from the tagged commit, manifest SHA recorded
- [ ] Env vars set (api column of §4); `doctor --service api` passes
- [ ] Start command `node dist/main.js`; `PORT` source confirmed
- [ ] `https://api.<domain>/health/ready` returns 200; TLS valid; CORS restricted to dashboard origin
- [ ] Restart policy = always; crash restart observed once in staging

**Bot**
- [ ] Env vars set (bot column); privileged intents match what is enabled in the Discord portal (none by default)
- [ ] Commands registered via the one-off job
- [ ] Exactly one instance running; leader lock verified (start a second → stays passive)
- [ ] Ping interaction works in the production guild; bot permissions are least-privilege
- [ ] Restart/takeover tested

**Worker**
- [ ] Env vars set (worker column; no Discord token)
- [ ] At least one instance running; outbox relay active (publish a test event → consumed)
- [ ] Timers/reconcilers running; DLQ empty
- [ ] Graceful shutdown test: SIGTERM during a job → job completes

**PostgreSQL**
- [ ] Version, TLS, connection limit documented; pool sizes within budget
- [ ] Migration + app roles separated (or limitation documented)
- [ ] Advisory locks verified through the chosen connection path
- [ ] Migrations applied with `migrate deploy`; drift check clean
- [ ] Append-only triggers/REVOKEs verified by `doctor`

**Redis**
- [ ] TLS, `noeviction` verified; prefix isolation confirmed
- [ ] Redis stop/start test: no business-data loss, queues recover

**Storage**
- [ ] Driver chosen (`s3` preferred); bucket private; encryption verified (stored bytes ≠ plaintext)
- [ ] If `local`: persistent disk verified and shared by api + worker
- [ ] Upload/download test through the API; no public URL possible

**Environment variables & secrets**
- [ ] All values in the host secret store; none in Git, artifacts, logs, or chat
- [ ] Keyring + audit signing key backed up separately
- [ ] Rotation procedure documented and `rotate-keys` tested on staging

**Ports & health**
- [ ] No hardcoded ports; env-driven only
- [ ] `/health/live` and `/health/ready` verified per service; `/detail` token-protected

**Persistent storage, logs, backups**
- [ ] Persistent paths (if any) outside the artifact directory
- [ ] Logs readable and retained for the required period (or forwarded)
- [ ] Backup job running; **restore test completed once** before go-live

**Migrations & rollback**
- [ ] Pre-migration snapshot taken
- [ ] Previous 3 artifacts retained per service; rollback rehearsed on staging

**Restart & shutdown**
- [ ] Restart policy verified; shutdown timeout < platform grace period

**Monitoring**
- [ ] Uptime monitor on API and dashboard
- [ ] Alerts configured for health, restarts, DLQ, outbox lag, audit integrity, backups
- [ ] Alert delivery tested to Discord **and** an external channel
