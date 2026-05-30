# 1. Stack — chosen technologies and rationale

Per-layer technology choices, with rationale grounded in the architecture's constraints. Where you've already chosen, the choice is **Confirmed**; where I've proposed a lean for you to confirm or override, it's marked **Open (Dn)**.

| Section | Topic |
|---|---|
| 1.1 | Backend language and framework |
| 1.2 | Database |
| 1.3 | Frontend (Admin UI) |
| 1.4 | Identity provider (IdP) |
| 1.5 | Object storage (media) |
| 1.6 | Async work and message transport |
| 1.7 | Observability |
| 1.8 | CI/CD and hosting |
| 1.9 | Tooling |
| 1.10 | Indonesia / locale specifics |
| 1.11 | Summary of open decisions |

---

## 1.1 Backend language and framework

**Confirmed:** TypeScript + Node.js (LTS).
**Confirmed:** NestJS as the application framework.

### Why TypeScript + Node.js

- **Shared types across frontend and backend.** Domain types (`Money`, `LocalizedText`, state enums) live in a shared package; frontend and backend speak the same vocabulary. This directly supports the handoff's "domain language over technical language" principle ([§8.2 of CLAUDE.md](../CLAUDE.md)).
- **Type system enforces module boundaries.** TypeScript's structural typing + ESM module resolution lets us forbid cross-context internal-type imports at build time (per [§5.3.7](../handoff/05-cross-cutting.md)).
- **Mature ecosystem** for OIDC (`openid-client`), Postgres (`pg`, Drizzle, Prisma), background jobs, and Beckn-friendly JSON tooling.

### Why NestJS

- **Module system aligns with bounded contexts.** Each context becomes a Nest `Module` with its own controllers (Interface Layer), services (Application Layer), and repository ports. Cross-context imports go through explicit `provider` injections — matching the ports + adapters pattern of [§2.4](../handoff/02-principles.md) of the handoff.
- **DI primitives** make ports + adapters cheap: define an abstract token (`AuthorizationPort`), inject by token, bind to a concrete adapter at the module level. Swapping adapters (in-process → RPC) is a module-level configuration change.
- **Built-in support for multiple transports** (HTTP, gRPC, message queues) without restructuring app code — useful if the Bridge later runs as a separate process or if we add a message bus.
- **Decorator-based pipes / guards** map cleanly to `requireCapability` ([§5.1.4 of handoff](../handoff/05-cross-cutting.md)) and idempotency-key handling ([§5.4](../handoff/05-cross-cutting.md)).

### Alternatives considered

- **Fastify + custom DI (tsyringe / awilix)** — lighter, more flexible, but requires hand-wiring the module-per-context discipline. More setup; less guardrails.
- **Hono** — edge-first, very lightweight; pricier in DI / structure for an app of this size.

Pick NestJS unless the team has strong reasons against decorators (some teams hate them); the tradeoff for the project's size and structure favors it.

---

## 1.2 Database

**Confirmed:** PostgreSQL.
**Open (D1):** Query / migration layer — propose **Drizzle ORM** with **`drizzle-kit`** for migrations.

### Why Postgres

The architecture **requires** an RDB with strong transactional semantics ([§5.2 outbox](../handoff/05-cross-cutting.md), [§5.3 inbox dedup](../handoff/05-cross-cutting.md), [§5.4 idempotency](../handoff/05-cross-cutting.md)). All three patterns write to a `_records` table **in the same transaction** as the state mutation. Postgres delivers this cheaply; document stores don't.

Postgres also gives:
- **`LISTEN`/`NOTIFY`** as a no-extra-infra in-process event dispatcher for v1 monolith (per [§1.6](#16-async-work-and-message-transport)).
- **`SELECT ... FOR UPDATE SKIP LOCKED`** for outbox-dispatcher claim semantics.
- **Per-column encryption** (`pgcrypto`) if [§5.6.9](../handoff/05-cross-cutting.md) field-level encryption is needed later.
- **JSONB** for the audit `source_envelope` ([§4.7 of handoff](../handoff/04-bounded-contexts/4.7-audit.md)).

### Why Drizzle (proposed — D1)

- **SQL-first, type-safe.** What you write looks like SQL, what TS sees is fully typed. Lower magic budget than Prisma; closer to the actual queries.
- **Schema lives in code** as TypeScript declarations; migrations are generated from schema diffs.
- **No generated client / no separate generation step at runtime** — friendlier in monorepos and CI.
- **First-class support for transactions, JSONB, custom column types** (needed for `Money`, `LocalizedText`, enums).

### Alternatives considered (D1)

| Option | Pros | Cons |
|---|---|---|
| **Prisma** | Largest community, great DX, clear docs | Magic generated client; weaker for complex queries; runtime overhead per migration |
| **MikroORM** | Unit-of-Work + Identity Map (DDD-friendly) | Heavier mental model; smaller community |
| **TypeORM** | Default in older NestJS examples | Known issues with active record vs data mapper confusion; slowing maintenance |
| **Raw SQL + Kysely** | Maximum control | More boilerplate; ORM features re-implemented |

If the team has Prisma muscle memory, switch — Drizzle's edge is transparency, not capability.

---

## 1.3 Frontend (Admin UI)

**Confirmed:** React + **TanStack Router** + **Vite**.

**Recommended pairings:**
- **Server state:** TanStack Query (natural pair with TanStack Router).
- **Tables:** TanStack Table.
- **Forms:** TanStack Form (or React Hook Form — D2a).
- **Schema validation:** Zod (shared with backend).
- **Styling:** Tailwind CSS (lean).
- **Component primitives:** **Open (D2)** — lean **shadcn/ui** (Radix-backed; you own the source).

### Why this stack

- **Type-safe routing** end-to-end. TanStack Router infers route params, search params, and loaders into types — the URL contract becomes part of the type system. Especially valuable for the URL-encoded active-Org / active-Store hierarchy from [§5.1.5 of handoff](../handoff/05-cross-cutting.md): `/orgs/<org-slug>/stores/<store-slug>/...`.
- **No Next.js / SSR overhead.** The Admin UI is authenticated-only and doesn't need SEO or RSC; Vite SPA is simpler to develop and deploy ([ADR-0021](../decisions/0021-pure-bpp-no-storefront.md) makes this an authenticated-only product anyway).
- **TanStack family is internally consistent** — Router, Query, Table, Form all share the same authoring style and TS philosophy.
- **Zod schemas shared with backend** validate form input on the client and request bodies on the server — one source of truth.

### Open (D2) — Component library

| Option | Why consider |
|---|---|
| **shadcn/ui** (proposed) | You own the source. Tailwind-native. Radix primitives underneath. No lock-in. Indonesian / RTL not needed. |
| **Mantine** | Batteries-included; data-grid; forms. Heavier install. |
| **Ark UI / Chakra v3** | Newer; Chakra v3 rebuilt on Ark. Tailwind-friendly. |
| **Custom** | Maximum control; large up-front cost. |

### Open (D2a) — Forms

| Option | Why consider |
|---|---|
| **TanStack Form** (proposed) | Aligns with rest of TanStack stack; type-safe; smaller community |
| **React Hook Form** | Largest community; mature; Zod resolver |
| **Conform** | Tighter Zod-first integration; smaller community |

---

## 1.4 Identity provider (IdP)

**Confirmed:** OIDC integration ([ADR-0009](../decisions/0009-identity-and-external-idp.md)) — provider is configurable.

**Open (D3):** Concrete provider for v1.

### Provider recommendation matrix

| Option | Pros for v1 | Cons |
|---|---|---|
| **Supabase Auth** (lean if D6 is Supabase Postgres) | Cheap; fast DX; OIDC-compatible; good Indonesian presence | Tighter coupling if we also use Supabase DB; some advanced features are paid |
| **Clerk** | Best-in-class DX; pre-built sign-in components | Pricier at scale; vendor lock for UX components |
| **Cognito** (lean if hosting on AWS) | AWS-native; cheap | Worst DX of the bunch |
| **Auth0** | Most mature, enterprise standard | Most expensive; overkill for v1 |
| **Keycloak** (self-host) | No vendor lock; full control | Operational burden; not ideal for v1 |

**Lean:** Supabase Auth if DB also lands on Supabase (D6); otherwise Clerk for fastest DX. Both honor the ADR-0009 OIDC-agnostic architecture, so this is reversible.

### Implementation notes (regardless of provider)

- Use `openid-client` (the de-facto OIDC library for Node).
- Map IdP claims to the User entity per [§4.1 of handoff](../handoff/04-bounded-contexts/4.1-identity.md):
  - `sub` → `external_subject_id`
  - `email`, `email_verified` → `email`, `email_verified_at`
  - `name` → `display_name`
  - `picture` → `avatar_url` (initial value only; mutable thereafter)
  - `locale` → `preferred_locale` (initial value only; mutable thereafter)
- Session is **our own opaque token** independent of IdP tokens ([§4.1](../handoff/04-bounded-contexts/4.1-identity.md)). Session storage: a `sessions` table in Postgres + signed cookie carrying the opaque session ID.

---

## 1.5 Object storage (media)

**Open (D4):** S3-compatible object store.

| Option | Lean? | Why |
|---|---|---|
| **Cloudflare R2** | Lean | Cheap; zero egress fees; S3-compatible API; good for catalog images (high read, predictable load) |
| **AWS S3** | If hosting on AWS | Battle-tested; high egress cost |
| **Backblaze B2** | Budget alternative | Cheap; S3-compatible; smaller ecosystem |
| **Supabase Storage** | Lean if D6 = Supabase | Tightly coupled to Supabase; simple |

### Implementation notes

- Domain knows only **references** ([§4.3 of handoff](../handoff/04-bounded-contexts/4.3-catalog.md)). Object storage is purely Infrastructure.
- Upload flow: admin UI requests a signed URL from backend; uploads directly to storage; backend records the reference.
- Image processing (resize, format conversion) is operational: either on-the-fly via Cloudflare Images (if R2) or via a worker.

---

## 1.6 Async work and message transport

### v1 — Postgres-only (Confirmed)

Per [§5.2 of handoff](../handoff/05-cross-cutting.md) the substrate is the **transactional outbox + dispatcher** pattern. For v1 in a modular monolith ([§6.3 of handoff](../handoff/06-operational.md)) this is **entirely Postgres-resident**:

- **Outbox table** per emitting context.
- **Inbox / `processed_events` table** per subscribing context.
- **Dispatcher** runs in-process as a background worker (NestJS scheduled task or a separate process consuming the same DB).
- **Wake mechanism**: Postgres `LISTEN`/`NOTIFY` from the outbox-write transaction (best-effort signal) + interval polling fallback.
- **Claim semantics**: `SELECT ... FOR UPDATE SKIP LOCKED` lets multiple dispatcher instances run safely.

No external bus in v1.

### Phase 2 (deferred)

When a context is extracted to its own service ([§6.3.3 of handoff](../handoff/06-operational.md)) or when scale demands it, introduce an external bus. The outbox + envelope contract stays the same; only the transport changes — that's the topology-neutrality promise of [§5.2.8](../handoff/05-cross-cutting.md).

Lean for when this happens: **Redis Streams** (if Redis is already in the stack for caching/sessions) or **NATS** (if not).

### Scheduled work (cron-style)

- NestJS `@Schedule` decorators are fine for v1 (in-process).
- Operational jobs needing cron: outbox pruning, inbox pruning, idempotency-record TTL cleanup, audit retention sweep, session expiry, stuck-events sweep.

---

## 1.7 Observability

**Confirmed approach:** OpenTelemetry SDK for traces and metrics; structured JSON logs.

**Open (D5):** Observability backend.

| Option | Lean? | Why |
|---|---|---|
| **Grafana Cloud** | Lean | Free tier covers v1; OTel-native; logs (Loki) + metrics (Mimir) + traces (Tempo) in one |
| **Datadog** | If team has it | Best-in-class; pricey |
| **Self-hosted (Loki + Tempo + Prometheus + Grafana)** | If team has K8s ops | No vendor; ops burden |
| **AWS CloudWatch + X-Ray** | If hosting on AWS | Adequate; AWS-locked |

### Implementation notes

- **Redacting logger** ([§5.6.6 of handoff](../handoff/05-cross-cutting.md)) is a wrapper around `pino` (proposed) or `winston` that consumes the PII catalog from `design/pii.md`.
- **`correlation_id` propagation**: set on the Nest request context; passed to outbox rows; propagates to subscribers via the envelope.
- **OTel auto-instrumentation** for Postgres, HTTP, and the IdP SDK comes free; manual spans around use-case boundaries.

---

## 1.8 CI/CD and hosting

### CI/CD — **Confirmed:** GitHub Actions

- Runs typecheck, lint, test (unit + integration via Testcontainers), build.
- Required checks on PR before merge: typecheck, lint, test, build.
- Workflows live in `.github/workflows/` of the code repo (separate from this design repo).

### Hosting — **Open (D6) Backend + (D7) DB + (D8) Frontend**

| Layer | Lean | Alternatives |
|---|---|---|
| **Backend** (D6) | **Fly.io** | Railway, Render, AWS ECS, GCP Cloud Run, self-managed K8s |
| **Database** (D7) | **Neon** or **Supabase** | RDS, Cloud SQL, self-managed Postgres |
| **Frontend** (D8) | **Cloudflare Pages** or **Vercel** | Netlify, Fly.io static, S3+CloudFront |

### Lean rationale (proposals)

- **Fly.io for backend**: simple Dockerfile-based deploy; good for monolithic Node services; Indonesian (Singapore) region available; cheaper than ECS for v1 traffic.
- **Neon for DB**: serverless Postgres, generous free tier, branching for preview environments. Supabase if D3 (IdP) is also Supabase.
- **Cloudflare Pages for frontend**: cheap; integrates with R2 (D4 lean); Workers available for edge logic if needed.

The architecture is hosting-neutral. These leans are about minimizing v1 ops surface, not a permanent commitment.

---

## 1.9 Tooling

| Tool | Choice | Why |
|---|---|---|
| **Package manager** | **pnpm** | Fast, disk-efficient, monorepo-native via workspaces |
| **Monorepo** | **pnpm workspaces** + **Turborepo** (Open D9) | Workspaces for resolution; Turbo for build orchestration if needed |
| **Linting** | **ESLint** + **typescript-eslint** | Standard |
| **Formatting** | **Prettier** | Standard; defer to ESLint for rules that overlap |
| **Type checking** | **`tsc` --noEmit** in CI | Authoritative |
| **Unit tests** | **Vitest** | Fast, ESM-native, Jest-compatible API |
| **Integration tests** | **Vitest + Testcontainers** | Real Postgres per test suite; clean teardown |
| **API contract tests** | **Vitest + Supertest** | Standard for HTTP layer |
| **Pre-commit hooks** | **lint-staged** + **husky** | Format + lint on staged files |
| **API documentation** | **NestJS Swagger module** for OpenAPI | Lives next to controllers |
| **Build** | **Vite** (frontend) + **swc/tsup** (backend) | Fast builds; ESM-first |

### Open (D9) — Turborepo or Nx or neither

| Option | Why consider |
|---|---|
| **Turborepo** (proposed) | Lightweight; minimal config; good caching |
| **Nx** | Heavier; better for very large monorepos; generators |
| **Neither** | pnpm workspaces + npm scripts only; simplest |

For v1 monorepo size (one app + a few shared packages), **Turborepo** is a good middle ground. Skip if the team prefers raw npm scripts.

---

## 1.10 Indonesia / locale specifics

Per [§5.7 of handoff](../handoff/05-cross-cutting.md) the platform default locale is `id` (Bahasa Indonesia) and `LocalizedText` is a domain value object.

### Implementation notes

- **`LocalizedText`** as a TS branded type: `{ entries: Record<BCP47Tag, string> }` with a runtime invariant (Zod `superRefine`) requiring the `id` key.
- **Currency**: `Money` as `{ amount: bigint; currency: ISO4217Code }`. Display via `Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR' })`.
- **Timezone**: store all timestamps as UTC; display in `Asia/Jakarta` (operational, not domain).
- **Date library**: native `Date` is workable; if more arithmetic needed, **`date-fns`** with `id` locale, or wait for Temporal (Node 22+ flagged).
- **Data residency**: Indonesia-resident DB ([§5.6.10 of handoff](../handoff/05-cross-cutting.md)). Singapore is the nearest region for most providers; Indonesia-region is available on AWS Jakarta (`ap-southeast-3`) and GCP Jakarta (`asia-southeast2`). Verify with hosting choice (D6, D7).
- **PDP compliance** (UU 27/2022): the scrub-in-place pattern from [§5.6 of handoff](../handoff/05-cross-cutting.md) is the implementation hook. Encryption at rest must be operational baseline.

---

## 1.11 Summary of open decisions

| D# | Topic | Section | Lean |
|---|---|---|---|
| D1 | Query / migration layer | [1.2](#12-database) | Drizzle ORM + `drizzle-kit` |
| D2 | Component library | [1.3](#13-frontend-admin-ui) | shadcn/ui |
| D2a | Form library | [1.3](#13-frontend-admin-ui) | TanStack Form |
| D3 | IdP provider | [1.4](#14-identity-provider-idp) | Supabase Auth (if DB is Supabase) or Clerk |
| D4 | Object storage | [1.5](#15-object-storage-media) | Cloudflare R2 |
| D5 | Observability backend | [1.7](#17-observability) | Grafana Cloud |
| D6 | Backend hosting | [1.8](#18-cicd-and-hosting) | Fly.io |
| D7 | Database hosting | [1.8](#18-cicd-and-hosting) | Neon (or Supabase if D3 = Supabase Auth) |
| D8 | Frontend hosting | [1.8](#18-cicd-and-hosting) | Cloudflare Pages (or Vercel) |
| D9 | Monorepo orchestration | [1.9](#19-tooling) | Turborepo |

When confirmed, each lean becomes the body text; the alternative table becomes "alternatives considered" if useful, or deleted if not.

---

## What's next

Once stack decisions land, the next doc to draft is [`02-repo-layout.md`](02-repo-layout.md) — the concrete directory structure and how it enforces the bounded-context boundaries from the handoff. After that, [`05-phases.md`](05-phases.md) for the implementation roadmap.
