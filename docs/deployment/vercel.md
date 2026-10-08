# Deploying the DevLand Dashboard on Vercel

Scope: `apps/dashboard` only. The dashboard is a Next.js app that **never touches the database or any backend secret**. It talks only to `apps/api` (hosted separately, see `wispbyte.md`). `packages/ui`, `packages/i18n`, `packages/contracts` and `packages/types` are bundled into the dashboard build; backend packages are never imported (enforced by dependency-cruiser, runbook step 3).

Status: planning document, **not yet executed**. Vercel settings names below reflect Vercel's standard monorepo workflow; re-check them against the current Vercel dashboard/docs when you create the project, since UI labels change.

## 1. Project setup

| Setting | Value |
|---|---|
| Framework preset | Next.js |
| Root Directory | `apps/dashboard` |
| "Include source files outside of the Root Directory" | **Enabled** (the app imports workspace packages) |
| Install command | `pnpm install --frozen-lockfile` (run at repo root by Vercel's monorepo support; pnpm version pinned by the root `packageManager` field) |
| Build command | `pnpm turbo run build --filter=@devland/dashboard...` |
| Output | Next.js default (`.next`) |
| Node.js version | same as `.nvmrc` (D4) |
| Git branch | `main` = Production; all other branches = Preview |
| Ignored Build Step | `npx turbo-ignore @devland/dashboard` (skip builds when only backend code changed) |

`next.config.ts` must set `transpilePackages: ['@devland/ui', '@devland/i18n', '@devland/contracts', '@devland/types']` (decision D1: `ui` is consumed from source so Tailwind can scan it).

## 2. Environment variables (Vercel project settings, per environment)

Only these. **No database URL, bot token, crypto keys, signing keys, Redis URL or storage credentials may ever exist in the Vercel project.**

| Variable | Public? | Production | Preview | Purpose |
|---|---|---|---|---|
| `API_BASE_URL` | server only | `https://api.<domain>` | **staging API URL only** | BFF proxy target (D2) |
| `DASHBOARD_ORIGIN` | server only | `https://dashboard.<domain>` | preview URL pattern | CSRF/Origin checks |
| `NEXT_PUBLIC_APP_ENV` | public | `production` | `preview` | UI environment banner |
| `NEXT_PUBLIC_DEFAULT_LOCALE` | public | `ar` | `ar` | default locale |
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY` | public | when push is added (later phase) | — | web push public key |

Rules: anything prefixed `NEXT_PUBLIC_` is visible to every visitor, so it must be non-secret. CI bundle scan (runbook step 22) fails the build if `DISCORD_BOT_TOKEN`, `CRYPTO_KEYS`, `DATABASE_URL` or similar strings appear in the client bundle. **Preview deployments must point at the staging API and a staging Discord application — never production.**

## 3. Authentication through the BFF proxy (decision D2)

- The browser only talks to the dashboard origin. Next.js route handlers/rewrites forward `/api/*` to `API_BASE_URL`.
- The API's `Set-Cookie` for `__Host-dl_session` is passed through, making the cookie **host-only on the dashboard origin** (HttpOnly, Secure, SameSite=Lax, Path=/, no Domain). This is why no parent-domain cookie is needed.
- Discord OAuth redirect URI = the **dashboard** callback route (`https://dashboard.<domain>/auth/callback`), which hands the code to the API through the proxy. Register exact redirect URIs per environment in the matching Discord application (production, staging; previews use a fixed staging alias domain, not random preview URLs, because Discord requires exact URIs).
- Verify on a Vercel preview that `Set-Cookie` survives the proxy and that the API sees the original client IP via `X-Forwarded-For` (needed for IP hashing; configure the API's trusted-proxy setting accordingly).
- If the proxy approach fails a requirement, write an ADR before switching to a shared-parent-domain cookie (`__Secure-` prefix with `Domain=.<domain>`), which weakens host isolation.

## 4. Domain and TLS
- `dashboard.<domain>` added as a custom domain; DNS per Vercel's instructions; TLS issued and renewed by Vercel.
- Redirect apex/`www` as you prefer; HSTS set via headers once the domain is stable.
- Monitor certificate and domain expiry (alerts in `wispbyte.md` §15 apply to this URL too).

## 5. Security headers (set in `next.config.ts`)
`Content-Security-Policy` with per-request nonces (no `unsafe-inline` scripts; `connect-src 'self'`, since API calls go through the same-origin proxy; `frame-ancestors 'none'`), `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal, `Cross-Origin-Opener-Policy: same-origin`. Playwright tests assert them (runbook step 22).

## 6. PWA behavior on Vercel
- Service worker serves the offline shell and static assets only. Authenticated API responses are **network-only, never cached** (tested in runbook step 23).
- Service worker file must be served with `Cache-Control: no-cache` so updates propagate; manifest and icons are static assets.
- Push notifications arrive in a later phase and need API-side VAPID keys (secret stays on the API, never in Vercel).

## 7. Health and monitoring
- `GET /api/health` on the dashboard returns build SHA and whether the configured API is reachable (`API_BASE_URL/health/ready`), without exposing internals.
- External uptime monitor on `https://dashboard.<domain>` and `/api/health`.
- Enable Vercel's deployment notifications to the Discord ops channel; track error rates via your chosen logging/analytics (optional `SENTRY_DSN` is server-side only).

## 8. Deployment flow
1. PR opened → CI (typecheck, lint, tests, Playwright) → Vercel Preview (staging API).
2. Merge to `main` → Production deployment builds automatically.
3. Order with backend releases: **backend first, dashboard last** (the dashboard must never expect an API route that is not live yet). Contract changes follow the same expand/contract discipline as the database: the API stays backward compatible with the previous dashboard build.
4. After deploy: login smoke test (OAuth → 2FA → shell), sessions page, logout.

## 9. Rollback
- Vercel **Instant Rollback** (promote the previous production deployment) from the Vercel dashboard.
- Before rolling back, confirm the previous dashboard build is compatible with the **current** API version (this is why API changes must be backward compatible for at least one release).
- Environment variable changes are not rolled back by redeploying; revert them explicitly.
- After rollback: smoke test, then fix forward on a branch.

## 10. What is and is not deployed to Vercel

| Deployed (via build) | Not deployed |
|---|---|
| `apps/dashboard` + bundled `ui`, `i18n`, `contracts`, `types` | `api`, `bot`, `worker`, `db`, `core`, `auth`, `access`, `audit`, `events`, `storage`, `payments`, `infra` |
| Static assets, fonts (self-hosted via `next/font`), manifest, service worker | `docs/`, tests, `testing`, `tooling`, `infrastructure/*`, `.github`, any `.env*` |

## 11. Checklist
- [ ] Vercel project created with Root Directory `apps/dashboard` and external-source inclusion enabled
- [ ] Build command and Ignored Build Step set; first build green
- [ ] Only the §2 variables exist; no backend secrets present
- [ ] Production env points to production API; Preview env points to staging API only
- [ ] Discord OAuth redirect URIs registered per environment (exact match)
- [ ] BFF proxy verified: cookie host-only, HttpOnly, Secure; client IP forwarded correctly
- [ ] Custom domain + TLS active; HSTS enabled after verification
- [ ] Security headers verified (CSP nonce, frame-ancestors, nosniff, HSTS)
- [ ] Bundle scan shows no secrets; CI check enabled
- [ ] PWA: offline shell works; authenticated responses never cached
- [ ] `/api/health` reachable; uptime monitor configured
- [ ] Rollback rehearsed once on a preview/staging project
- [ ] Release order documented: backend first, dashboard last
