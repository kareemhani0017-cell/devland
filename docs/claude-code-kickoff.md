# Claude Code kickoff prompt — DevLand Phase 2

Paste everything inside the block into Claude Code, opened at the root of your GitHub repo (after copying the `devland-phase1/` files into it: `docs/`, `packages/db/prisma/schema.prisma`).

```text
You are executing Phase 2 of DevLand Enterprise in this repository.

Read first, in order: CLAUDE.md (create it from docs/05-config-checklist.md §5 if missing),
docs/01-architecture.md … docs/05-config-checklist.md, docs/phase-2-runbook.md,
docs/deployment/wispbyte.md, docs/deployment/vercel.md.

Execute docs/phase-2-runbook.md steps 1–28 in order, honoring §0 and §0a (hard constraints).
Approved decisions: D1 CommonJS, D2 same-origin BFF, D3 Prisma client generated in the production build,
D5 `pnpm deploy --prod` (verify real pnpm behavior). D4 is DEFERRED: do not pin Node (.nvmrc/engines/CI)
until WispByte's supported Node version is verified with evidence. Do NOT set Prisma binaryTargets.

Rules:
1. Run each step's commands for real. Record command, exit code, trimmed output, date and git SHA in
   docs/phase-2-report.md. Never claim a command passed unless you ran it. Record failures and skips honestly.
2. One commit per step on a feature branch. Do not proceed past a failing step; fix inside the step.
   If a Phase 1 decision must change, write an ADR in docs/adr/ and stop to ask me.
3. No secrets in the repo. Only .env.example files with empty values. Never print secret values.
4. No business domains (store, orders, tickets, payment logic, delivery, reviews). No Phase 3.
5. WispByte-specific code stays behind adapters; do not write a wispbyte driver. Treat all WispByte
   capabilities as unverified; fill docs/deployment/wispbyte.md §1 only with real evidence I give you.
6. Artifacts must be marked "provisional": true until D4/D5 and the Prisma host target are verified.
7. When step 28 is done, STOP and report: exact files changed, final folder tree, Prisma validation,
   migration result (clean apply + drift check), typecheck, lint, tests (counts), build, security checks
   (gitleaks, pnpm audit, bundle secret scan), deployment artifact results, WispByte capability
   verification status, unresolved issues, and an exact Phase 3 proposal. Wait for my approval.

If anything blocks you (missing Docker, no network, ambiguous requirement), stop and tell me rather than guessing.
```

## Information you need to provide to Claude Code

| Needed for | What |
|---|---|
| Step 1 / D4 | WispByte's supported Node version(s), with a docs link or screenshot |
| Step 4 / D3 | Whether the repo uses Prisma 6 or 7 (let it check the installed version) |
| Steps 18, 21 | A fresh Discord **dev** application token and a test guild ID, in a local git-ignored `.env.local` only |
| Steps 22–23 | A decision on Arabic font/brand direction if you have one |
| Deployment | WispByte answers to `wispbyte.md` §1 (process count, ports, stop signal, DB/Redis/storage offerings, host OS/OpenSSL for Prisma) |
