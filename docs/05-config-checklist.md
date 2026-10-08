# DevLand Enterprise — Phase 1, Part 5: Environment config, checklist, open items

Covers deliverables 10, 19, plus spec coverage and open decisions.

## 1. Environment variable design (deliverable 10)

Rules: one zod schema in `packages/config` is the only reader of `process.env`; the app **refuses to boot** on invalid/missing values; secrets never have defaults; each environment has its own values and its own Discord application. `.env.example` is committed; real `.env*` files are git-ignored.

```dotenv
# ---------- Core ----------
NODE_ENV=development                 # development | staging | production
APP_ENV=development                  # drives feature/sandbox rules; must match deployment
LOG_LEVEL=info
SERVICE_NAME=api                     # api | bot | worker (set per process)
DEFAULT_LOCALE=ar                    # ar | en
OWNER_DISCORD_ID=                    # bootstrap owner (not a secret)

# ---------- Discord ----------
DISCORD_APPLICATION_ID=
DISCORD_BOT_TOKEN=                   # SECRET
DISCORD_CLIENT_SECRET=               # SECRET (dashboard OAuth2)
DISCORD_GUILD_ID=                    # the DevLand guild (single-guild bot)
DISCORD_OAUTH_REDIRECT_URI=http://localhost:4000/auth/discord/callback

# ---------- Database (adapter: postgres) ----------
DATABASE_URL=                        # pooled connection (app)
DIRECT_DATABASE_URL=                 # direct connection (migrations)
DB_SUPPORTS_ADVISORY_LOCKS=true
DB_SUPPORTS_ROLE_REVOKE=false        # true only if app role is separate and restricted

# ---------- Redis (adapter: redis) ----------
REDIS_URL=
REDIS_KEY_PREFIX=devland:dev:        # per-environment isolation if Redis is shared (avoid sharing prod)

# ---------- Storage (adapter: storage) ----------
STORAGE_DRIVER=local                 # local | s3 | wispbyte (wispbyte only after API confirmed)
STORAGE_BUCKET=devland-dev
STORAGE_LOCAL_PATH=./.data/storage   # local driver only
S3_ENDPOINT=                         # s3 driver (also MinIO)
S3_REGION=
S3_ACCESS_KEY_ID=                    # SECRET
S3_SECRET_ACCESS_KEY=                # SECRET
STORAGE_MAX_UPLOAD_MB=25
STORAGE_TEMP_URL_TTL_SEC=300

# ---------- Crypto / keys ----------
CRYPTO_KEYS={"k1":"<base64-32-bytes>"}   # SECRET keyring: id -> AES-256 key
CRYPTO_ACTIVE_KEY_ID=k1
AUDIT_SIGNING_KEY=                   # SECRET, kept apart from DB credentials
IP_HASH_PEPPER=                      # SECRET
SECRET_PROVIDER=env                  # env | vault (vault later)

# ---------- Auth / sessions ----------
SESSION_COOKIE_NAME=__Host-dl_session
SESSION_IDLE_MINUTES=30
SESSION_ABSOLUTE_HOURS=12
STEP_UP_WINDOW_MINUTES=5
TOTP_ISSUER=DevLand
DASHBOARD_ORIGIN=http://localhost:3000
API_BASE_URL=http://localhost:4000
CSRF_SECRET=                         # SECRET

# ---------- Queues / events ----------
OUTBOX_POLL_INTERVAL_MS=500
OUTBOX_BATCH_SIZE=100
EVENT_MAX_ATTEMPTS=8
WORKER_QUEUES=all                    # or comma list for split deployments
BOT_LEADER_LOCK_TTL_SEC=30

# ---------- Rate limiting ----------
RATE_LIMIT_USER_PER_MIN=60
RATE_LIMIT_IP_PER_MIN=120

# ---------- Observability ----------
SENTRY_DSN=                          # optional
OTEL_EXPORTER_OTLP_ENDPOINT=         # later phase
HEALTH_TOKEN=                        # SECRET for detailed health endpoint
```

**Not env, but DB-configured by the Owner** (so changes are audited): exchange rate, commission rules, order/capacity limits (default 10 per developer), claim reminder timings, change-request limits, queue behaviour, payment methods, role mapping, channel bindings, policies, SLA, feature flags. Env holds only infrastructure and secrets; business config lives in `Setting` & friends.

**Secrets handling:** `SECRET_PROVIDER=env` reads from the host's secret store/environment in Phase 2. "Independent vault, dynamic secrets, automatic rotation" are only available once a real vault exists; the interface is ready, the claim is not made yet.

## 2. Spec coverage matrix (nothing silently dropped)

| Spec § | Topic | Status after Phase 1 |
|---|---|---|
| 1–4 | Identity, architecture, stack, domains | Designed (docs 01) |
| 5–6 | `/setup`, interaction model | Schema + design ready (`DiscordChannelBinding/Panel`); build in MVP |
| 7–8 | Tickets/orders, statuses | Schema + state machine designed; MVP |
| 9–10 | Store, currency | Schema designed; MVP |
| 11 | Payments | Schema + abstraction designed; MVP (manual proof) |
| 12–15 | Developers, commission, assignment, deadlines | Schema + rules designed; MVP |
| 16 | Delivery | Schema designed; MVP |
| 17–18 | Queue, order limits | Schema (`QueueEntry`, settings) designed; MVP |
| 19–20 | My Orders, customer levels | My Orders MVP; levels schema only |
| 21–25 | Discounts, loyalty, referrals, gift cards, wallet | Hooks in schema; wallet tables ready; later phases |
| 26 | Refunds | `Refund` table + approval model; later |
| 27 | Invoices | `Invoice` table; Discord invoice MVP-lite, PDF later |
| 28 | Reviews | Schema; MVP |
| 29 | Support | `Ticket` support fields; later |
| 30–33 | Applications, suggestions, giveaways, announcements | Later |
| 34–36 | Moderation, welcome, autorole | Later |
| 37–40 | KB, feedback, AI | Later (event hooks defined) |
| 41–44 | Policies, verification, contracts, disputes | Policies MVP-lite; verification levels in `User`; contracts/disputes later |
| 45, 50–51 | Anti-fraud, threat detection, abuse | Signals in schema (`SecurityEvent`, proof hash); engine later |
| 46–49 | Security, RBAC/ABAC, JIT, rule engine | RBAC + ABAC + grants designed (doc 03); JIT flow + rule engine later |
| 52, 97 | Forensic audit | Designed (doc 03 §4); MVP implements chain + checkpoints |
| 53–54, 69, 85–86 | Incidents, self-healing, status, observability, recovery | Reconcilers + health in MVP; rest later |
| 55 | Event platform | Designed (doc 04); outbox/inbox/DLQ in MVP |
| 56 | Workflow engine | Later (events are its triggers) |
| 57 | Notifications | `Notification` + dedupe MVP; quiet hours/push later |
| 58–60 | Owner workspace, developer workspace, balance | Developer balance/withdrawal MVP-lite; workspaces grow per phase |
| 61–63 | Analytics, search, reporting | Later (ledgers/events are the sources) |
| 64–68 | API platform, keys, dev platform, toolkit, integrations | Later |
| 70 | Approval engine | `ApprovalRequest/Decision` in schema; simple policies first |
| 71–76, 95 | Storage, backups, retention, DB/encryption/secrets | Storage + encryption MVP; backups/retention later; DB owner-only rule honored (no SQL console) |
| 77–84 | Environments, git, testing, sandbox, flags, releases, maintenance | Env/CI designed (doc 01 §6); flags table ready; rest per phase |
| 87–94 | Domains, dashboard design/auth, PWA, mobile, lockdown, recovery, AI policy | Auth designed; dashboard MVP basic; PWA/lockdown/recovery later |
| 96, 98–103 | Integrity, errors, logging, config | Designed and enforced from MVP |
| 100–101 | Discord logging, log search | Channel routing MVP-lite; search later |
| 104, 109 | Owner control center, dashboard pages | Basic Owner Dashboard in MVP; pages added per phase |

## 3. Phase 1 implementation checklist (deliverable 19)

**Delivered in this Phase 1 package**
- [x] Architecture + diagram, process model, hard rules
- [x] Monorepo structure, module boundaries and ownership
- [x] Prisma schema (core + platform tables) and ERD
- [x] Raw-SQL constraint plan (partial uniques, append-only triggers)
- [x] Permission/RBAC/ABAC model, catalog subset, visibility profiles
- [x] Event bus design, envelope, MVP event catalog, example schemas
- [x] Queue/worker design, reconcilers
- [x] Storage and payment abstractions
- [x] Security architecture, threat model summary
- [x] Auth/session architecture
- [x] Audit/forensic architecture with honest guarantee table
- [x] Env variable design, deployment matrix, dependency list
- [x] Order state machine and claim-reminder behaviour

**First tasks of Phase 2 (only after your approval)**
1. Scaffold repo: pnpm workspace, Turborepo, TS base config, ESLint/Prettier, dependency-cruiser boundary rules, `CLAUDE.md`, CI skeleton, CODEOWNERS.
2. `packages/config` (zod env) + `pnpm doctor`.
3. `packages/db`: `prisma validate/format`, initial migration, raw SQL migration, seed (permissions, default roles, default `Setting`s, default services, default commission rule).
4. `packages/infra`: clock/ids, crypto (envelope + keyring), cache/queue wrappers, storage interface + `local` and `s3` drivers + contract tests.
5. `core/audit` + `core/events` (UnitOfWork, outbox relay, inbox, DLQ) with tests including chain verification.
6. `core/access` + `core/auth` (mapping, AuthorizationService, sessions, TOTP).
7. Bot foundation (leader lock, interaction router, i18n AR/EN, `/setup` skeleton) and dashboard foundation (auth shell, design tokens, RTL).

Then MVP order per your list: store → orders/tickets → payments + proof review → pools/claim → deadlines → delivery/change requests → reviews → basic Owner Dashboard.

## 4. Open decisions and assumptions I made (please confirm or change)

1. **Claimable after payment approval**, not before (doc 04 §5).
2. **`CANCELLED` status added** to your status list.
3. **One deadline proposal per developer per order** by default (your rule), further rounds need Owner permission.
4. **IP addresses stored hashed** by default (privacy-first); say so if you need raw IPs for investigations.
5. **Dashboard 2FA = TOTP** first; passkeys later.
6. **Order ↔ ticket numbering share one sequence** for order tickets; support tickets use their own (`DL-S-…`).
7. **WispByte driver deferred** until you tell me what it exposes (Postgres? Redis? storage API, SFTP, S3-compatible?). Until then dev = local/MinIO, and any S3-compatible or standard Postgres/Redis host works.
8. **Prisma version:** schema uses 6.x syntax; if you want Prisma 7 I will adjust the datasource config.
9. Decimal money with a `Money` value object (not integer minor units). Tell me if you prefer integer piastres/cents.

## 5. Suggested `CLAUDE.md` for the repo (for Claude Code)

```md
# DevLand — working agreement
- Read docs/01..05 before changing architecture. Update docs in the same PR as behaviour changes.
- Business logic ONLY in packages/core/modules/<domain>/{domain,application}. Bot handlers, controllers, workers are thin adapters.
- A module touches only its own tables; cross-module = exported service interface or events.
- Every mutation: one transaction = state change + AuditEvent + OutboxEvent. Use UnitOfWork; never call prisma.$transaction ad hoc.
- Money: Money value object (Decimal). No floats. Currency always explicit.
- IDs/amounts in events are strings/uuids. No secrets, proofs or PII in event payloads.
- Never hardcode Discord role/channel IDs, URLs, secrets. Config via packages/config or Setting table.
- Never create Discord roles from the bot.
- Never claim immutable/encrypted/HA unless the adapter capability says so; update doctor output + docs.
- Add/modify an event => add zod schema in packages/contracts/events and bump version if breaking.
- Tests: unit for domain rules/state machines; integration (Testcontainers) for any service touching the DB; idempotency + concurrency tests for payments, claims, delivery acceptance, ledger.
- Commands: pnpm i · pnpm dev · pnpm test · pnpm typecheck · pnpm lint · pnpm prisma:migrate · pnpm doctor
- Stop after each phase and wait for approval.
```
