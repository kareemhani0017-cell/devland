# DevLand Enterprise — Phase 1, Part 1: Architecture

Status: Phase 1 design (no business logic yet). Covers deliverables 1, 2, 3, 15, 16, 17, 18.

## 1. Architectural stance

**Modular monolith, deployed as three Node processes plus a dashboard**, sharing one domain core. This is "microservices-ready" without paying the microservice tax for a single-guild business.

| Process | What it is | Scales how |
|---|---|---|
| `apps/api` | NestJS (Fastify adapter). REST API, auth, SSE. The only HTTP surface the dashboard talks to. | Horizontally (stateless; sessions in DB + Redis cache) |
| `apps/bot` | NestJS standalone app + discord.js gateway client. **Thin**: parses interactions, calls application services, renders embeds. | Singleton (one guild, one gateway connection; leader lock in Redis prevents double-run) |
| `apps/worker` | NestJS standalone app. BullMQ consumers, outbox relay, schedulers, reconcilers. | Horizontally per queue |
| `apps/dashboard` | Next.js (Vercel-friendly). Never touches the DB; talks only to `apps/api`. | Vercel |

**Hard rules (enforced by lint + dependency-cruiser in CI):**
1. Business logic lives only in `packages/core` modules. Bot handlers, controllers and workers are adapters that call them.
2. A module may read/write only its own tables. Cross-module needs go through the other module's exported service interface or through events.
3. State change + outbox event + audit event are written in **one DB transaction**.
4. The dashboard has no direct database access.
5. Infrastructure (Postgres, Redis, storage, secrets, payments) is reached only through interfaces in `packages/infra`.

### Why not real microservices now
Your spec says "avoid unnecessary microservice complexity." Domain boundaries are enforced in code and DB ownership, so any module (payments, notifications, AI) can later be lifted into its own service by swapping its in-process interface for an HTTP/queue client. Heavy independent workloads (PDF generation, backups, audit verification, AI) are already isolated as worker queues.

## 2. High-level diagram

```mermaid
flowchart LR
  subgraph Clients
    DC[Discord users<br/>customers / developers / owner]
    DASH[Owner Dashboard<br/>Next.js PWA]
    EXT[3rd-party API clients<br/>later phase]
  end

  subgraph Edge
    DISCORD[(Discord Gateway & REST)]
  end

  subgraph DevLand["DevLand runtime (pnpm monorepo)"]
    BOT[apps/bot<br/>discord.js adapter]
    API[apps/api<br/>NestJS REST + SSE]
    WRK[apps/worker<br/>BullMQ consumers + schedulers]

    subgraph CORE["packages/core — domain modules"]
      direction TB
      M1[orders / tickets / store]
      M2[payments / invoices]
      M3[developers / delivery / deadlines]
      M4[auth / access / audit / events]
      M5[notifications / settings / setup]
    end
  end

  subgraph Infra["packages/infra — adapters (swappable)"]
    PG[(PostgreSQL<br/>source of truth)]
    RD[(Redis<br/>queues, cache, locks, rate limits)]
    ST[(Object storage<br/>wispbyte / s3 / local)]
    SEC[Secret provider<br/>env now, vault later]
    PAY[Payment providers<br/>manual-proof now, gateways later]
  end

  DC <--> DISCORD <--> BOT
  DASH -->|HTTPS + session cookie| API
  EXT -.->|API keys / OAuth2 (later)| API
  BOT --> CORE
  API --> CORE
  WRK --> CORE
  CORE --> PG
  CORE --> RD
  CORE --> ST
  CORE --> SEC
  CORE --> PAY
  PG -. outbox rows .-> WRK
  WRK -->|discord-outbound queue| BOT
```

**Event flow in one sentence:** a command mutates state → same transaction writes `OutboxEvent` + `AuditEvent` → outbox relay (worker) publishes to BullMQ → consumers (notifications, Discord renderers, timers, analytics) process idempotently via `InboxEvent`. Discord messages are a *projection*; PostgreSQL is the source of truth.

## 3. Monorepo structure (pnpm workspaces + Turborepo)

```
devland/
├─ CLAUDE.md                     # conventions for Claude Code (see docs/05)
├─ package.json  pnpm-workspace.yaml  turbo.json  tsconfig.base.json
├─ .github/
│  ├─ CODEOWNERS
│  └─ workflows/ ci.yml  security.yml  release.yml  migrate.yml
├─ docker-compose.dev.yml        # postgres, redis, minio, mailpit
├─ docs/                         # these documents + ADRs (docs/adr/)
├─ apps/
│  ├─ api/        src/{main.ts, app.module.ts, http/{controllers,guards,interceptors,filters}, sse/}
│  ├─ bot/        src/{main.ts, gateway/, interactions/{commands,buttons,selects,modals}, renderers/, i18n/}
│  ├─ worker/     src/{main.ts, queues/, processors/, schedulers/, reconcilers/}
│  └─ dashboard/  src/{app/, components/, features/, lib/api-client/, styles/}
├─ packages/
│  ├─ config/     zod env schema + typed ConfigService (single source of config)
│  ├─ contracts/  zod schemas shared by api/bot/worker/dashboard
│  │              ├─ events/<event-name>/v1.ts   (schema registry)
│  │              ├─ api/                         (request/response DTOs)
│  │              └─ permissions.ts               (permission catalog)
│  ├─ db/         prisma/{schema.prisma, migrations/, sql/, seed/}  + tx helper, base repository
│  ├─ core/       src/modules/<domain>/
│  │              ├─ <domain>.module.ts
│  │              ├─ application/   services (public API of the module)
│  │              ├─ domain/        entities, state machines, pure rules (no I/O)
│  │              ├─ infrastructure/ repositories (Prisma), mappers
│  │              ├─ events/        emitted + consumed handlers
│  │              └─ index.ts       ONLY exports the public interface
│  ├─ infra/      src/{storage, cache, queue, secrets, crypto, payments, clock, ids}/
│  ├─ i18n/       ar.json / en.json + typed t() helpers
│  ├─ ui/         dashboard design system (tokens, components, GSAP presets)
│  ├─ testing/    factories, test containers, fake providers, sandbox seed
│  └─ tooling/    eslint-config, tsconfig presets, dependency-cruiser rules
└─ scripts/       seed, doctor, verify-audit-chain, generate-event-docs
```

## 4. Domain / module boundaries

Each module owns its tables. "Phase" = when it is implemented; tables for later modules are added by additive migrations so nothing in Phase 1 blocks them.

| Module | Owns (tables) | Public surface (used by others) | Emits | Phase |
|---|---|---|---|---|
| `config` | Setting, FeatureFlag, ExchangeRate | `SettingsService.get/set`, `ExchangeRateService.current()` | `config.*`, `exchange_rate.changed` | MVP |
| `audit` | AuditEvent, AuditCheckpoint, AuditVerificationRun | `AuditService.record()` (tx-aware) | `audit.integrity_violation_detected` | MVP |
| `events` | OutboxEvent, InboxEvent, DeadLetterEvent, IdempotencyRecord | `EventBus.publish(tx, evt)`, `@OnEvent` consumers | — | MVP |
| `auth` | User (identity part), Session, TwoFactorCredential, RecoveryCode, TrustedDevice, SecurityEvent | `SessionService`, `IdentityService.resolveDiscordUser()` | `auth.*` | MVP (2FA minimal) |
| `access` | Permission, AccessRole, AccessRolePermission, AccessGrant, ApprovalRequest/Decision | `AuthorizationService.can(ctx, action, resource)` | `access.*`, `approval.*` | MVP (RBAC + basic ABAC; JIT later) |
| `customers` | CustomerProfile, CustomerLevel | `CustomerService` | `customer.*` | MVP (profile only) |
| `developers` | DeveloperProfile, DeveloperPoolMembership, CommissionRule, DeveloperLedgerEntry, WithdrawalRequest | `DeveloperService`, `CommissionResolver`, `DeveloperBalanceService` | `developer.*`, `withdrawal.*` | MVP |
| `store` | Service, ServiceQuestion, FeatureGroup, ServiceFeature, FeatureDependency, FeatureIncompatibility | `CatalogService`, `PricingService.quote()` (pure) | `store.*` | MVP |
| `orders` | Order, OrderItem, OrderStatusHistory, OrderAssignment, DeveloperRequest, OrderNote/Link/Attachment, QueueEntry, Sequence | `OrderService` + order state machine | `order.*` | MVP |
| `tickets` | Ticket | `TicketService` (Discord channel lifecycle via bot port) | `ticket.*` | MVP (order tickets; support later) |
| `payments` | PaymentMethod, Payment, PaymentProof | `PaymentService`, `PaymentProvider` port | `payment.*` | MVP |
| `deadlines` | DeadlineProposal, DeadlineExtension | `DeadlineService` | `deadline.*` | MVP |
| `delivery` | Delivery, DeliveryFile, ChangeRequest | `DeliveryService` | `delivery.*`, `change_request.*` | MVP |
| `reviews` | Review | `ReviewService` | `review.submitted` | MVP |
| `invoices` | Invoice | `InvoiceService` | `invoice.*` | MVP-lite (Discord invoice; PDF later) |
| `policies` | Policy, PolicyVersion, PolicyAcceptance | `PolicyService.acceptFor(order)` | `policy.*` | MVP-lite (single terms policy) |
| `storage` | FileObject | `FileService` (never exposes raw provider) | `file.*` | MVP |
| `notifications` | Notification | `NotificationService.enqueue()` | — | MVP (Discord/DM) |
| `discord-setup` | DiscordChannelBinding, DiscordPanel | `SetupService.run()` (idempotent) | `setup.completed` | MVP |
| `wallet`, `refunds` | Wallet, WalletTransaction, Refund | `WalletService`, `RefundService` | `wallet.*`, `refund.*` | Later (schema ready) |
| `support`, `kb`, `ai`, `loyalty`, `referrals`, `giftcards`, `promotions`, `disputes`, `contracts`, `antifraud`, `incidents`, `workflows`, `api-platform`, `integrations`, `status`, `analytics`, `reporting`, `backups`, `maintenance`, `releases`, `moderation`, `giveaways`, `suggestions`, `applications`, `announcements`, `welcome`, `autorole` | added per phase | — | — | Later |

**Cross-module rules of thumb.** `orders` never reads `payments` tables; it reacts to `payment.approved`. `delivery` never edits `Order.status`; it calls `OrderService.transition()`. Pricing is a pure function in `store` so it can be unit-tested exhaustively and reused for quote display, order creation and (later) the AI price suggestion.

## 5. Infrastructure adapters (no assumptions about WispByte)

Each adapter has: an interface, a config-selected driver, a contract test suite every driver must pass, and a documented capability matrix. **Drivers shipped in Phase 2: `local`/dev drivers + standard drivers. A `wispbyte` driver is written only after you confirm what WispByte actually exposes** (I will not guess its API).

### 5.1 PostgreSQL
- Prisma connects via `DATABASE_URL` (pooled) and `DIRECT_DATABASE_URL` (migrations). Works with any managed or self-hosted Postgres ≥ 15.
- Capability flags in config: `DB_SUPPORTS_ADVISORY_LOCKS` (default true), `DB_SUPPORTS_ROLE_REVOKE` (needed to harden audit table).
- If the host cannot give separate DB roles, audit hardening falls back to triggers only and the limitation is reported by `pnpm doctor`.

### 5.2 Redis
Used for: BullMQ, rate limiting, locks, session cache, bot leader lock. **Nothing business-critical lives only in Redis.** Timers and reminders are reconstructable from Postgres by reconcilers (see doc 04), so a Redis flush degrades latency, not correctness. BullMQ needs `maxmemory-policy noeviction`; `doctor` checks it and warns loudly if the provider forbids it.

### 5.3 Storage abstraction (deliverable 15)

```ts
interface StorageProvider {
  readonly id: string;                       // 'local' | 's3' | 'wispbyte'
  readonly capabilities: {
    presignedUrls: boolean; serverSideEncryption: boolean;
    objectVersioning: boolean; objectLock: boolean;   // objectLock = real WORM
  };
  put(key: string, body: Readable | Buffer, meta: PutMeta): Promise<PutResult>;
  getStream(key: string): Promise<Readable>;
  head(key: string): Promise<ObjectHead | null>;
  delete(key: string): Promise<void>;
  createTemporaryUrl?(key: string, ttlSec: number, opts: UrlOpts): Promise<string>; // only if capability
}
```
Rules implemented in `FileService` (provider-independent, so they hold even on a weak provider):
- **Encrypt before upload** (envelope encryption: random per-file AES-256-GCM data key, wrapped by a keyring key; `FileObject.encryptionKeyId/encryptedDataKey`). Provider-side encryption is a bonus, not relied upon.
- **Integrity:** SHA-256 of plaintext stored in `FileObject.sha256`, verified on every read.
- **Access:** no public buckets, ever. If the provider cannot presign, files are streamed through `apps/api` behind a permission check + short-lived HMAC token. Payment proofs are `PAYMENT_PROOF` class: only roles with `payment.proof.view` (Owner) and the paying customer.
- **Versioning:** at application level (new `FileObject` per upload; delivery versions reference their own files), not dependent on provider versioning.
- **WORM:** only claimed when `capabilities.objectLock` is true. Otherwise audit/evidence rely on the hash chain (tamper-evident) and the doctor reports "WORM: unavailable".

### 5.4 Payment provider abstraction (deliverable 16)

```ts
interface PaymentProvider {
  readonly key: string;                                  // 'manual_proof' now; 'paypal_gateway', 'stripe'... later
  readonly capabilities: { requiresProof: boolean; supportsWebhooks: boolean; supportsRefunds: boolean };
  describe(method: PaymentMethod, order: OrderSnapshot, locale: Locale): PaymentInstructions;  // what the customer sees
  initiate?(payment: Payment): Promise<InitiateResult>;                 // gateways
  verify?(payment: Payment): Promise<VerificationResult>;               // gateways
  handleWebhook?(raw: RawWebhook): Promise<WebhookOutcome>;             // signed, idempotent
  refund?(payment: Payment, amount: Money): Promise<RefundResult>;      // gateways
}
```
MVP registers one implementation, `manual_proof`, used by PayPal (manual transfer), Vodafone Cash and InstaPay. Adding a real gateway later = new provider class + a `PaymentMethod.providerKey` change; **order/payment state machines and the review UI do not change.**

**Payment state machine** (`domain/payment.state.ts`, pure, exhaustively tested):

```
PENDING_PROOF → PROOF_SUBMITTED → UNDER_REVIEW → APPROVED → (PARTIALLY_REFUNDED | REFUNDED)
                      ↑                 └→ REJECTED → PENDING_PROOF (customer re-uploads; old proof SUPERSEDED)
any pre-APPROVED → CANCELLED
```
Integrity: approval runs `SELECT … FOR UPDATE` on the payment, checks `version`, and is protected by the raw-SQL partial unique index "one APPROVED payment per order". A double-click on Approve returns the same result via `IdempotencyRecord`.

## 6. Deployment architecture (deliverable 17)

| | Development | Staging | Production |
|---|---|---|---|
| Purpose | local + Claude Code | pre-release, mirrors prod | live DevLand server |
| Discord | **separate test application + test guild** | **separate application + staging guild** | production application + DevLand guild |
| Postgres / Redis | docker-compose | separate instances | separate instances |
| Storage | `local` or MinIO | separate bucket/path + keys | separate bucket/path + keys |
| Secrets | `.env.local` (git-ignored) | CI/host secret store | host secret store (+ vault when available) |
| Payments | sandbox data only | sandbox only | real manual-proof methods |
| Data | seeded fake data | anonymized or synthetic only | real |
| Deploy | local | auto on merge to `main` | manual approval gate on tagged release |

- **Process layout:** `api`, `bot`, `worker` as three services on the backend host (WispByte if it can run long-lived Node processes and reach Postgres/Redis; otherwise any VPS/PaaS — the code does not care). `dashboard` on Vercel with `API_BASE_URL` pointed at `api.<domain>`.
- **Migrations:** `prisma migrate deploy` runs as a separate CI job **before** new app versions start; expand/contract pattern (add → backfill → switch → drop in a later release) so rollbacks stay safe. Pre-migration snapshot/backup is a required step in the production pipeline.
- **Domains:** `dashboard.` (Vercel), `api.` (backend host + TLS), `cdn.` and `status.` reserved for later phases.
- **Honest limits to document in the runbook:** single-region, single bot instance (no failover for the Discord gateway — Discord only allows one session per shard anyway; recovery = fast restart + reconcilers replaying missed work), HA only as good as the chosen Postgres/Redis providers.
- **CI gates:** typecheck, lint, dependency-cruiser boundaries, unit + integration (Testcontainers), `prisma validate`, migration dry-run on a fresh DB, secret scan (gitleaks), dependency audit, event-schema compatibility check.

## 7. Dependency list (deliverable 18)

Package names only; **pin the current stable major at install time and commit the lockfile** (I cannot verify latest versions from here, and I will not invent version numbers). Use `pnpm dlx npm-check-updates` / Renovate afterwards.

**Workspace/tooling:** `pnpm`, `turbo`, `typescript`, `eslint` + `typescript-eslint`, `prettier`, `dependency-cruiser`, `vitest`, `husky` + `lint-staged`, `commitlint`, `changesets` (release notes/semver), `gitleaks` (CI).

**api:** `@nestjs/common` `@nestjs/core` `@nestjs/platform-fastify` `@nestjs/config` `@nestjs/swagger` `@nestjs/terminus` `@nestjs/schedule`(light cron only) · `fastify` `@fastify/cookie` `@fastify/helmet` `@fastify/rate-limit` `@fastify/cors` · `zod` + `nestjs-zod` · `nestjs-pino` `pino` · `otplib` (TOTP) · `argon2` · `undici` · `rxjs`.

**bot:** `discord.js` · `@nestjs/core` (standalone context) · `pino`.

**worker:** `bullmq` · `ioredis` · `@nestjs/core` (standalone) · `pdfkit` or headless-chromium (invoice PDF, later).

**db:** `prisma` + `@prisma/client` (pin per the Prisma-version note in the schema header) · `pg` (raw SQL helpers).

**infra:** `@aws-sdk/client-s3` (+ `@aws-sdk/s3-request-presigner`) for the S3 driver · `ssh2-sftp-client` only if WispByte turns out to be SFTP-only · Node built-in `crypto` (AES-256-GCM, HMAC, SHA-256) — no custom crypto.

**dashboard:** `next` `react` `react-dom` · `tailwindcss` · `gsap` (+ `@gsap/react`) · `@tanstack/react-query` · `@tanstack/react-table` · `recharts` (or `visx`) · `next-intl` (AR/EN, RTL) · `zod` · `react-hook-form` · `@serwist/next` (PWA/service worker) · `lucide-react`.

**testing:** `vitest` `supertest` `@testcontainers/postgresql` `testcontainers` `@faker-js/faker` `playwright` (E2E) · `k6` (load, later).

**observability (Phase 1-light):** `pino` structured logs now; `@opentelemetry/*` + `prom-client` wired in a later phase behind the same logger/tracer interfaces.
