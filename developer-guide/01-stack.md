# 1. Stack — requirements and reference choices

This BPP is planned to land inside a larger monorepo (see `developer-guide/README.md` → "Monorepo context"). The monorepo doesn't currently dictate a specific framework or tool, so this section is structured in two layers:

- **Required:** characteristics the implementation MUST have, derived from the [handoff](../handoff/README.md) and the [ADRs](../decisions/). These don't move regardless of which framework / library is eventually used.
- **Lean (v1):** a concrete reference choice that satisfies the requirements — useful as a starting point for a standalone v1 deploy. May be superseded by monorepo conventions when the merger lands.

Read the **Required** lines first. The **Lean** lines are proposals you can keep, swap, or drop.

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

### Required

- **Language with a strong static type system.** Module boundaries between bounded contexts ([§2.4 of handoff](../handoff/02-principles.md), [§5.3.7](../handoff/05-cross-cutting.md)) must be enforceable at build time; runtime-only checks aren't sufficient.
- **Capability for `LocalizedText`, `Money`, and other domain value objects to be expressed precisely.** The language needs first-class struct / record / branded-string types (per [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md), [ADR-0008](../decisions/0008-localization-and-localizedtext.md)).
- **Ecosystem for OIDC, RDB transactions, and JSON-heavy protocol work.** Beckn is JSON-over-HTTP with signatures; the chosen runtime must have mature libraries for these.
- **HTTP framework supporting middleware composition** for the cross-cutting concerns: `requireCapability` ([§5.1.4 of handoff](../handoff/05-cross-cutting.md)), idempotency keys ([§5.4](../handoff/05-cross-cutting.md)), `correlation_id` propagation ([§5.2.1](../handoff/05-cross-cutting.md)).
- **Request validation that integrates with the domain's value-object definitions.** Schemas authored once should validate inbound requests, type outbound responses, and ideally be shared with the frontend.
- **Dependency-injection capability** (built into the framework or composable as a separate library). Ports + adapters ([§2.4 of handoff](../handoff/02-principles.md)) require swapping concrete implementations at the module level.

### Lean (v1)

- **Language**: TypeScript on Node.js (LTS).
  - *Why a fit*: same-language fullstack (shared types between Admin UI and BPP); structural type system + ESM modules enforce context boundaries at build time; mature ecosystem for everything above.
- **HTTP framework**: **Hono** (subject to monorepo convention).
  - *Why a fit*: lightweight; middleware composition for the cross-cutting concerns; `@hono/zod-validator` + `@hono/zod-openapi` integrate Zod schemas as the single source of truth for validation + OpenAPI; type-safe client generation via Hono RPC pairs cleanly with the frontend.
- **DI container**: **awilix** (proxy injection, no decorators) — **Confirmed (v1)**.
  - *Why*: per-request scoping is a hard requirement for `correlation_id`, `active_org_id`, `active_store_id`, actor (per [§5.1.5](../handoff/05-cross-cutting.md), [§5.2.1](../handoff/05-cross-cutting.md)); awilix has first-class scoped containers. Explicit registration + no `reflect-metadata` aligns with the project's explicit-over-magic posture.
  - Alternatives considered: **tsyringe** (decorator-based; familiar to NestJS users), **no container** (constructor wiring grows linearly with context count). The pattern these implement — looking up a port by token, binding it to a concrete adapter at module load, swapping in tests — is the same regardless of library, so D0 is replaceable later if the monorepo dictates otherwise.

Other reasonable framework choices that satisfy the requirements: **Fastify**, **Express + ts-rest**, **NestJS** (heavier but ships DI + modules built in). If the monorepo standardizes on one of these, that supersedes the Hono lean — the architecture and the patterns translate.

### Enforcement of architectural rules (regardless of framework)

The handoff's discipline points need somewhere to live in code:

| Concern | Where it lives |
|---|---|
| Bounded-context boundaries | **Folder structure + lint rules.** ESLint `import/no-restricted-paths` or `eslint-plugin-boundaries` forbids cross-context internal-type imports at build time. See [`02-repo-layout.md`](02-repo-layout.md) (when drafted). |
| Ports + adapters wiring | **DI container** (D0). Adapters registered at app startup; ports injected into use cases by token. |
| Application-Layer use-case envelope | A thin in-house abstraction: `validate → authorize → idempotency check → transaction → return`. ~50 lines per route avoided. Lives in a shared `kernel` or `application` package. |
| Outbox + inbox + idempotency table writes | Same transaction as the state mutation ([§5.2](../handoff/05-cross-cutting.md), [§5.3.6](../handoff/05-cross-cutting.md), [§5.4.4](../handoff/05-cross-cutting.md)). Implemented in the use-case envelope. |
| Authorization decisions | `requireCapability(name, scope)` invoked at the top of every mutating use case ([§5.1.4 of handoff](../handoff/05-cross-cutting.md)). Implemented as middleware or as the first line of the use case body. |
| `correlation_id` propagation | Per-request scope in the DI container; passed to outbox writes; carried into subscriber jobs. |

These belong to the architecture, not the framework. Whatever framework lands, these patterns survive.

---

## 1.2 Database

### Required

- **Relational database with ACID transactions.** The transactional outbox ([§5.2 of handoff](../handoff/05-cross-cutting.md)), inbox dedup ([§5.3.6](../handoff/05-cross-cutting.md)), and idempotency-key records ([§5.4.4](../handoff/05-cross-cutting.md)) all write a record **in the same transaction** as the state mutation. Document stores without multi-document transactions don't fit.
- **Strong typing for monetary values** (`Money` uses integer minor units per [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)) — the DB must support integer / decimal types that round-trip safely with the application.
- **JSON / JSONB column type** for `source_envelope` in audit records ([§4.7 of handoff](../handoff/04-bounded-contexts/4.7-audit.md)) and for `LocalizedText` value-object storage ([§5.7 of handoff](../handoff/05-cross-cutting.md)).
- **Row-level locking with skip semantics** for outbox-dispatcher claim (multiple dispatcher workers reading the same outbox without contention).
- **Schema migrations** as code-tracked, reviewable artifacts (not ad-hoc SQL).

### Lean (v1)

- **Engine**: **PostgreSQL**.
  - *Why a fit*: ACID; `JSONB`; `SELECT ... FOR UPDATE SKIP LOCKED`; `LISTEN`/`NOTIFY` for an in-process dispatcher; `pgcrypto` if field-level encryption is needed later ([§5.6.9 of handoff](../handoff/05-cross-cutting.md)).
  - This is the natural default. If the monorepo standardizes on a different RDB with the same capabilities (MySQL 8 with skip-locked, etc.), substitute.
- **Query / migration layer**: **Drizzle ORM** + `drizzle-kit` for migrations — **Confirmed (v1)**.
  - *Why*: SQL-first, type-safe, schema-as-code; no runtime client generation. Repository implementations in `contexts/<X>/infrastructure/repositories/` are written as SQL-shaped queries against typed schema — natural fit for the architecture's Domain ↔ Infrastructure separation. Migrations are plain SQL files generated from schema diffs, reviewable as SQL. Complex reads (Beckn catalog projection, Order Quote construction, Audit cross-context queries) benefit from the ability to write the SQL directly.
  - Alternatives considered: **Prisma** (largest community; generated client adds a layer that has to be bridged; less natural for complex reads); **Kysely** (purer query builder; thinner migration tooling); **MikroORM** (Unit-of-Work + Identity Map; heavier mental model). Any of these implements the same handoff-mandated patterns (outbox / inbox / idempotency in the same transaction as state) — the choice is about authoring style. If the monorepo standardizes on one of these, defer.

---

## 1.3 Frontend (Admin UI)

### Required

- **SPA suitable for an authenticated-only product.** No SEO; no public buyer-facing surface ([ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)). SSR / RSC are not requirements.
- **Type-safe routing** that can encode the active-Org and active-Store hierarchy from [§5.1.5 of handoff](../handoff/05-cross-cutting.md): `/orgs/<org-slug>/stores/<store-slug>/...`. URL params should be part of the type system.
- **Form library that integrates with the same validation schemas used by the backend** — single source of truth for shapes like `CreateProduct`, `UpdateStore`, etc.
- **Server-state management** distinct from form-state (cache, refetch, invalidation).
- **Component primitives** that don't impose a visual identity that conflicts with the monorepo's design system (if one exists).

### Lean (v1)

- **Library**: **React**.
- **Routing**: **TanStack Router** (type-safe params and loaders).
- **Build**: **Vite**.
- **Server state**: **TanStack Query**.
- **Tables**: **TanStack Table**.
- **Forms**: **React Hook Form** + `@hookform/resolvers/zod` — **Confirmed (v1)**.
  - *Why*: pairs out-of-the-box with shadcn/ui (its official `<Form>` examples build on RHF + Zod resolver); largest community; mature; same Zod schemas as the backend validate forms client-side.
  - Alternatives considered: **TanStack Form** (better typing, TanStack-family consistency — but requires extra wiring against shadcn/ui's defaults); **Conform** (Zod-first design; smaller community).
- **Schema validation**: **Zod** (shared with backend in a contracts package).
- **Styling**: **Tailwind CSS**.
- **Component primitives**: **shadcn/ui** — **Confirmed (v1)**.
  - *Why*: Tailwind-native (matches confirmed styling); Radix primitives underneath (best-in-class accessibility); you own the source (no version lock; full customization freedom; zero upgrade tax). Strongest fit with the confirmed stack.
  - Alternatives considered: **Mantine** (batteries-included DataTable / forms; has its own styling system that fights Tailwind); **Ark UI / Chakra v3** (newer headless primitives; Tailwind-friendly but less polished docs); **Custom** (maximum control; large upfront cost).

The TanStack family is internally consistent (Router / Query / Table share authoring style and TS philosophy). Forms deviate to RHF because of the shadcn/ui pairing. If the monorepo already standardizes on a different routing + data-fetching pair (e.g., Next.js + RSC, Remix, or React Router + custom data layer), substitute — the architectural requirements above don't change.

---

## 1.4 Identity provider (IdP)

### Required

- **OIDC integration** ([ADR-0009](../decisions/0009-identity-and-external-idp.md)). The platform delegates authentication entirely; no password storage in BPP.
- **Provider-agnostic adapter** so the concrete IdP can be swapped without touching the domain.
- **OIDC claims consumed**: `sub`, `email`, `email_verified`, `name`, optional `picture`, optional `locale`. The User entity maps these per [§4.1 of handoff](../handoff/04-bounded-contexts/4.1-identity.md).

### Lean (v1) — **Deferred to monorepo (D3)**

Concrete provider for v1, if standalone:

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

- Use a standard OIDC client library (in Node: `openid-client` is the de-facto choice).
- Claim → User mapping per [§4.1 of handoff](../handoff/04-bounded-contexts/4.1-identity.md):
  - `sub` → `external_subject_id`
  - `email`, `email_verified` → `email`, `email_verified_at`
  - `name` → `display_name`
  - `picture` → `avatar_url` (initial value only; mutable thereafter)
  - `locale` → `preferred_locale` (initial value only; mutable thereafter)
- Session is **the platform's own opaque token** independent of IdP tokens ([§4.1](../handoff/04-bounded-contexts/4.1-identity.md)). Storage: a `sessions` table in the RDB + signed cookie carrying the opaque session ID.

---

## 1.5 Object storage (media)

### Required

- **Object storage for catalog media** ([§4.3 of handoff](../handoff/04-bounded-contexts/4.3-catalog.md)). The domain holds only references; storage is Infrastructure.
- **Direct-upload flow** with signed URLs — the BPP doesn't proxy bytes through the application layer.
- **Indonesia-acceptable data residency** ([§5.6.10 of handoff](../handoff/05-cross-cutting.md)).

### Lean (v1) — **Deferred to monorepo (D4)**

| Option | Notes |
|---|---|
| **Cloudflare R2** (lean) | Cheap; zero egress fees; S3-compatible API; image-processing companion (Cloudflare Images) available |
| **AWS S3** | If hosting on AWS; battle-tested; egress cost considerations |
| **Backblaze B2** | Budget S3-compatible alternative |
| **Supabase Storage** | If D3 = Supabase Auth and/or D7 = Supabase Postgres |
| **Whatever the monorepo standardizes on** | Most monorepos already have an object-store adapter — reuse it |

---

## 1.6 Async work and message transport

### Required

- **Transactional outbox + asynchronous dispatcher** ([§5.2 of handoff](../handoff/05-cross-cutting.md)). Events written to a local outbox table in the same transaction as the state mutation; a dispatcher delivers them to subscribers.
- **Inbox dedup per subscription** ([§5.3.6](../handoff/05-cross-cutting.md)). Subscribers maintain a `processed_events` table keyed by `(subscription_name, event_id)`.
- **At-least-once delivery semantics**. Subscribers must dedup; producers must commit atomically.
- **Scheduled work** for operational jobs: outbox pruning, inbox pruning, idempotency-record TTL cleanup, audit retention sweep, session expiry, stuck-events sweep.

### Lean (v1) — RDB-resident

For v1 in a modular monolith ([§6.3 of handoff](../handoff/06-operational.md)), the entire substrate is RDB-resident:

- **Outbox table** per emitting context.
- **Inbox / `processed_events` table** per subscribing context.
- **Dispatcher** runs in-process as a background worker (or as a sibling process consuming the same DB).
- **Wake mechanism**: Postgres `LISTEN`/`NOTIFY` from the outbox-write transaction (best-effort signal) + interval polling fallback. Equivalent mechanism if a different RDB is used.
- **Claim semantics**: `SELECT ... FOR UPDATE SKIP LOCKED` lets multiple dispatcher instances run safely.

No external bus in v1.

### Phase 2 (deferred)

When a context is extracted to its own service ([§6.3.3 of handoff](../handoff/06-operational.md)) or when scale demands it, introduce an external bus. The outbox + envelope contract stays the same; only the transport changes — that's the topology-neutrality promise of [§5.2.8](../handoff/05-cross-cutting.md).

Candidates when this happens: Redis Streams (if Redis is already in the stack), NATS, or whatever the monorepo standardizes on.

### Scheduled work (cron-style)

Scheduled jobs run in a sibling worker process or under a job library.

| Option | Notes |
|---|---|
| **`node-cron` in a sibling worker process** (lean) | Trivial setup; lives in the same package; can share DI container with the HTTP app |
| **BullMQ** (Redis-backed) | Production-grade; retries; observability; needed once jobs get heavier |
| **Postgres-native (`pg_cron` extension)** | Zero new infrastructure if D7 supports it |
| **Whatever the monorepo uses** | If the monorepo has a scheduler convention (Temporal, Inngest, etc.), use it |

All operational jobs are small enough that the lean covers v1.

---

## 1.7 Observability

### Required

- **Three operational planes**: logs, metrics, traces ([§6.2 of handoff](../handoff/06-operational.md)). Audit is a fourth, *business* plane and stays separate from operational telemetry.
- **Structured logs** (JSON at the wire) with standard fields: `timestamp`, `level`, `service`, `context_name`, `correlation_id`, `causation_id`, `user_id` (where authorized to log), `active_org_id`, `active_store_id`.
- **PII-aware redacting logger** ([§5.6.6 of handoff](../handoff/05-cross-cutting.md)) — every logger goes through a redaction layer driven by the PII catalog in `design/pii.md`.
- **`correlation_id` propagation** across the entire request → use case → outbox → subscriber chain.
- **Tracing instrumentation** at use-case boundaries; standard library autoinstrumentation for HTTP / DB / IdP.

### Lean (v1)

- **Telemetry SDK**: **OpenTelemetry** (vendor-neutral; works with most backends).
- **Logger library**: **pino** (fast, structured, low overhead) — alternatives: bunyan, winston.
- **Backend** — **Deferred to monorepo (D5)**: see below.
- **`correlation_id` propagation**: set in a per-request DI scope; passed explicitly to outbox writes; included in the event envelope per [§5.2.1](../handoff/05-cross-cutting.md).

### Deferred (D5) — Observability backend

| Option | Notes |
|---|---|
| **Grafana Cloud** (lean) | Free tier covers v1; OTel-native; logs (Loki) + metrics (Mimir) + traces (Tempo) |
| **Datadog** | Best-in-class; pricey |
| **Self-hosted (Loki + Tempo + Prometheus + Grafana)** | No vendor; ops burden |
| **AWS CloudWatch + X-Ray** | If hosting on AWS |
| **Whatever the monorepo uses** | Defer if the monorepo has a standard |

---

## 1.8 CI/CD and hosting

### Required

- **CI pipeline** that runs typecheck, lint, test (unit + integration), and build on every PR; merge gated on green.
- **Per-environment isolation** for dev / staging / prod ([§6.5.3 of handoff](../handoff/06-operational.md)): distinct credentials for IdP, CDS, registry; production never accepts test-issued Beckn signatures.
- **Indonesia-acceptable data residency** for the DB ([§5.6.10 of handoff](../handoff/05-cross-cutting.md)).

### Lean (v1) — **subject to monorepo conventions**

| Layer | Lean | Notes |
|---|---|---|
| **CI** | GitHub Actions | Standard; defer if the monorepo uses another CI |
| **Backend hosting (D6)** | **Fly.io** | Simple Docker deploy; Singapore region |
| **Database hosting (D7)** | **Neon** or **Supabase** | Serverless Postgres; preview-environment branching (Neon) |
| **Frontend hosting (D8)** | **Cloudflare Pages** or **Vercel** | Static SPA hosting |

Alternatives are abundant: Railway, Render, AWS (ECS, RDS, CloudFront), GCP (Cloud Run, Cloud SQL), self-managed K8s. The architecture is hosting-neutral. **Most monorepos already have a deployment story** — reuse it. The leans above are only relevant for a standalone v1.

---

## 1.9 Tooling

> **Most of this section defers to monorepo conventions.** Listed leans assume a standalone v1; align with the host monorepo when it lands.

### Required

- **Type checking** in CI (`tsc --noEmit` or equivalent) — authoritative.
- **Linting with cross-package boundary rules** (`eslint-plugin-boundaries` or `import/no-restricted-paths`) to enforce bounded-context isolation per [§5.3.7 of handoff](../handoff/05-cross-cutting.md).
- **Unit and integration testing** with real RDB instances (e.g., via Testcontainers) for integration tests.
- **OpenAPI / schema-based contract** for the BPP's HTTP surface — generated from the same validation schemas used at runtime.
- **Type-safe API client** for the frontend — either generated from OpenAPI or via the chosen framework's RPC mechanism.

### Lean (v1)

| Tool | Lean | Notes |
|---|---|---|
| **Package manager** | pnpm | Fast; workspace-native |
| **Monorepo orchestration** | Turborepo — **Deferred to monorepo (D9)** | Lightweight task caching; defer to monorepo standard |
| **Linting** | ESLint + typescript-eslint + boundary plugin | Required (boundary plugin specifically) |
| **Formatting** | Prettier | Standard |
| **Type checking** | `tsc --noEmit` in CI | Required |
| **Unit tests** | Vitest | Fast, ESM-native, Jest-compatible |
| **Integration tests** | Vitest + Testcontainers | Real RDB per suite |
| **HTTP-layer tests** | Vitest + Supertest (or framework equivalent) | Standard |
| **Pre-commit hooks** | lint-staged + husky | Format + lint on staged files |
| **OpenAPI generation** | From the chosen framework's Zod-integration (e.g., `@hono/zod-openapi`) | Avoid hand-written OpenAPI |
| **Build** | Vite (frontend) + `tsup` or `tsx` (backend) | ESM-first |

### Deferred (D9) — Monorepo orchestration

| Option | Notes |
|---|---|
| **Turborepo** (lean) | Lightweight; good caching; minimal config |
| **Nx** | Heavier; better for very large monorepos; generators |
| **Neither (raw pnpm workspaces + npm scripts)** | Simplest |
| **Whatever the host monorepo uses** | Most likely — defer when the merger lands |

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

## 1.11 Decision status

### Confirmed for v1

Choices locked for the v1 build. Still subject to monorepo conventions when the merger lands.

| D# | Topic | Section | Choice |
|---|---|---|---|
| D0 | DI container | [1.1](#11-backend-language-and-framework) | **awilix** |
| D1 | Query / migration layer | [1.2](#12-database) | **Drizzle ORM** + `drizzle-kit` |
| D2 | Component library | [1.3](#13-frontend-admin-ui) | **shadcn/ui** |
| D2a | Form library | [1.3](#13-frontend-admin-ui) | **React Hook Form** + Zod resolver |

### Deferred to monorepo

Operational or ecosystem choices the host monorepo is likely to own. Leans noted inline are starting points if a standalone v1 ships before the merger; otherwise these resolve when the merger lands.

| D# | Topic | Section | Inline lean (if standalone v1) |
|---|---|---|---|
| D3 | IdP provider | [1.4](#14-identity-provider-idp) | Clerk or Supabase Auth |
| D4 | Object storage | [1.5](#15-object-storage-media) | Cloudflare R2 |
| D5 | Observability backend | [1.7](#17-observability) | Grafana Cloud |
| D6 | Backend hosting | [1.8](#18-cicd-and-hosting) | Fly.io |
| D7 | Database hosting | [1.8](#18-cicd-and-hosting) | Neon |
| D8 | Frontend hosting | [1.8](#18-cicd-and-hosting) | Cloudflare Pages or Vercel |
| D9 | Monorepo orchestration | [1.9](#19-tooling) | Turborepo |

### Still open

None — all per-decision picks for v1 are either Confirmed or Deferred to monorepo.

---

The **Required** items in each section don't move regardless of which leans land or which choices the monorepo imposes.

---

## What's next

- [`02-repo-layout.md`](02-repo-layout.md) — the BPP **package**'s internal structure within the host monorepo, and how it enforces the bounded-context boundaries from the handoff. Layout is largely portable; specific lint-rule paths depend on the monorepo's workspace structure.
- [`05-phases.md`](05-phases.md) — implementation roadmap (dependency-ordered build plan). Stack-agnostic.
