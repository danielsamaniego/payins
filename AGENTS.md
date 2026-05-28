# Kunfupay-Payins — Umbrella Agent Instructions

Top-level guidance for AI coding agents (Cursor, Codex, Claude Code, …) and humans
working across the two Payins repos. **This is an umbrella, not a monorepo.** Each
child repo is fully independent and has its own canonical `AGENTS.md`.

## Repos

| Repo | Path | Stack | Canonical agent doc |
|---|---|---|---|
| **Backend** | [`Kunfupay-Payins-Back/`](Kunfupay-Payins-Back/) | Hono, TS, PostgreSQL 16, Prisma 7, Redis (lock) | [`Kunfupay-Payins-Back/AGENTS.md`](Kunfupay-Payins-Back/AGENTS.md) |
| **Frontend** | [`Kunfupay-Payins-Front/`](Kunfupay-Payins-Front/) | Next.js 15, React 19, internal pnpm workspace (`apps/checkout`, `apps/dashboard`, `packages/*`) | [`Kunfupay-Payins-Front/AGENTS.md`](Kunfupay-Payins-Front/AGENTS.md) |

> **Read the child repo's `AGENTS.md` first** when working in that repo. This file
> only carries cross-repo rules — anything specific to backend or frontend lives in
> the child.

## What lives at this umbrella level

```
Kunfupay-Payins/                           git repo: danielsamaniego/payins
├── AGENTS.md, CLAUDE.md, README.md        Umbrella orientation (this directory).
├── PAYINS_SERVICE_PLAN.md                 Cross-repo roadmap, scope, phases.
├── .gitmodules                            Registers the two children as git submodules.
├── Kunfupay-Payins-Back/                  Submodule → danielsamaniego/payins-back.
│   └── docs/deployment.md                 Back-specific deploy notes (Vercel + Docker).
└── Kunfupay-Payins-Front/                 Submodule → danielsamaniego/payins-front.
    └── docs/deployment.md                 Front-specific deploy notes (Vercel + Docker).
```

The two children are **git submodules**, each pinned at a specific commit. Clone the
umbrella with `--recurse-submodules` to bring them down in one shot
(`git clone --recurse-submodules <umbrella-url>`); otherwise hydrate later with
`git submodule update --init --recursive`. To bump the pin to each child's latest
`main`: `git submodule update --remote --merge && git commit -am "bump submodules"`.

Each child repo has its own `package.json`, `pnpm-lock.yaml`, `.npmrc`, `biome.json`,
`Dockerfile`, `docker-compose*.yml`, `.husky/`, **and its own `docs/deployment.md`**
(deployment is per-repo, not cross-cutting), and (in the front) `pnpm-workspace.yaml`
+ `turbo.json`. **Do not introduce monorepo plumbing at this level** — no root
`package.json`, no `pnpm-workspace.yaml`, no `turbo.json`, no `docker-compose.*.yml`,
no umbrella `docs/`. The umbrella's only role is documentation routing + the
submodule registry.

## Canonical Source

- **`AGENTS.md` in each child repo is the canonical instruction file for that repo.**
- `CLAUDE.md` (in each location) is a thin wrapper that imports the local `AGENTS.md`
  (`@AGENTS.md`).
- **Language:** code and comments in English; docs in English. **Responses to the user
  are always in Spanish.** This applies in every repo.

## Service Snapshot

**Payins** is a standalone, multi-tenant payment-in orchestration platform. The
**backend** is a Hono service that orchestrates multiple external providers
(Stripe, Ebanx) across many methods (card, Pix, Boleto, PSE, Yape, Nequi, PLIN,
MercadoPago via Ebanx), countries, and currencies to execute one-time payments,
subscriptions (auto-provider, managed, reminder), payment links, refunds, and disputes.
Inbound provider webhooks are normalized into a stable, signed outbound event contract.

The **frontend** repo ships two Next.js apps: `apps/checkout` (public, payer-facing —
renders hosted / embedded UI per `FlowType` for the methods Payins integrates natively)
and `apps/dashboard` (authenticated superadmin console — methods, commissions,
platforms, observability; phase 2 adds reduced platform-scoped self-service).

The platform is **platform-agnostic**: no file references any concrete consumer product;
no business concept like "sale" or "order" exists in the domain. Integrators use an
opaque `reference` field for their own reconciliation.

## Universal Rules (apply to BOTH repos)

These rules cross the seam between the two repos. They are restated in each child's
`AGENTS.md`, but if the two ever drift this file wins.

- **Tenancy:** `Platform` is the tenant; it authenticates via API key. Every `Platform`
  owns one or more `Account`s (a default one is auto-created at registration). All
  account-scoped entities carry `platform_id` **and** `account_id` as `NOT NULL` FKs.
  No `mode` flag on Platform — simple vs. aggregator emerges from the number of
  Accounts. Every query is filtered by tenant; cross-tenant access fails in tests.
- **Credentials:** Never hardcode credentials. API keys: argon2id hash of the secret,
  public prefix visible. Provider credentials: libsodium sealed box at rest, master key
  in env/KMS, never logged.
- **Timestamps:** Unix milliseconds as `number` everywhere. In the DB they are
  `BigInt`. Keep a single representation end-to-end; no conversions.
- **Amounts:** Integer minor units as `BigInt` in the backend; `number` at the API
  boundary (`amount_minor: number` in JSON) and inside the fronts. Never use float or
  decimal for money. `$1.99 = 199`.
- **Percentages (commissions/markups):** basis points as integer (0–10000, where
  `10000 = 100%`). Never float. Half-even rounding when converting to integer minor
  units.
- **Entity IDs:** UUID v7 only, generated in application code through `IIDGenerator`.
  The DB never generates IDs. Never UUID v4.
- **Country:** ISO-3166-1 alpha-2 uppercase. **Currency:** ISO-4217 uppercase.
- **PCI:** Payins **never** accepts or stores PAN/CVV — only provider tokens. The
  provider's browser SDK (Stripe.js, Ebanx fields) tokenizes in the user's browser; the
  token is what hits the backend. CI gate (back repo):
  `rg -iE "pan|cvv|card.?number"` outside `src/provider-adapters/` must be empty.
- **Consumer isolation:** No file in either repo references a specific consumer
  product. CI gate: `rg -i "kunfupay|sale|order|product"` under source trees must be
  empty.

## Cross-repo workflow

The two repos are physically independent — there is no shared lockfile, no shared
workspace, no shared CI. Cross-cutting changes (backend ABI + a frontend) become **two
separate PRs**:

1. Ship the backend change first; it must remain backwards-compatible for at least one
   release.
2. Regenerate the frontend api-client from the new OpenAPI; ship the frontend PR.
3. Remove the backwards-compat shim in a later backend PR.

Reference flows that need to stay in sync:

- **API contract:** `Kunfupay-Payins-Back/src/.../routes` defines it via
  `hono-openapi` → emitted as `/openapi`. `Kunfupay-Payins-Front/packages/api-client`
  consumes that spec.
- **Banned terms / PCI gates:** same regex on both sides, enforced in each repo's own
  CI / pre-commit (see each `AGENTS.md`).

## Cross-cutting docs

- **Roadmap and phases:** [`PAYINS_SERVICE_PLAN.md`](PAYINS_SERVICE_PLAN.md) — applies
  to both repos.
- **Deployment (Vercel + Docker local):** scoped to each repo —
  [`Kunfupay-Payins-Back/docs/deployment.md`](Kunfupay-Payins-Back/docs/deployment.md)
  ·
  [`Kunfupay-Payins-Front/docs/deployment.md`](Kunfupay-Payins-Front/docs/deployment.md).
- Backend-specific deep dives: [`Kunfupay-Payins-Back/docs/`](Kunfupay-Payins-Back/docs/)
  (domain, datamodel, architecture, providers, integration-guide).
- Frontend-specific deep dives:
  [`Kunfupay-Payins-Front/docs/`](Kunfupay-Payins-Front/docs/)
  (canonical: `frontend-architecture.md`).

## Load Relevant Context First (umbrella router)

Before any task, read the relevant files below. Do not implement before loading the
applicable context.

| Task area | Read first |
|-----------|------------|
| Anything in the backend | [`Kunfupay-Payins-Back/AGENTS.md`](Kunfupay-Payins-Back/AGENTS.md) → then the docs it points to |
| Anything in the frontend | [`Kunfupay-Payins-Front/AGENTS.md`](Kunfupay-Payins-Front/AGENTS.md) → then `Kunfupay-Payins-Front/docs/frontend-architecture.md` |
| Roadmap / scope / phases | [`PAYINS_SERVICE_PLAN.md`](PAYINS_SERVICE_PLAN.md) |
| Deployment of this or that repo | each repo's own `docs/deployment.md` — see [back](Kunfupay-Payins-Back/docs/deployment.md), [front](Kunfupay-Payins-Front/docs/deployment.md) |
| Universal contract rules | This file (§ Universal Rules) |
