# 3. Development setup

How to get the BPP package running locally — from a fresh clone to a working dev environment in under an hour. Stack-specific guidance uses the v1 leans (TypeScript + pnpm + Drizzle + Vitest + Vite); patterns translate.

| Section | Topic |
|---|---|
| 3.1 | Reading this section |
| 3.2 | Prerequisites |
| 3.3 | First-time setup |
| 3.4 | Running locally |
| 3.5 | Environment configuration |
| 3.6 | Database |
| 3.7 | IdP sandbox |
| 3.8 | Seed data |
| 3.9 | Common dev tasks |
| 3.10 | Troubleshooting |
| 3.11 | What this section doesn't cover |

---

## 3.1 Reading this section

This guide describes how to develop the BPP package locally. Some of it is **architecture-imposed** — every implementation needs Postgres (or equivalent), needs migrations, needs seed data, needs a way to develop frontend + backend together with shared types. Other bits are **stack-specific** to the v1 leans and would change if the monorepo standardizes on different tools.

When the host monorepo dictates dev tooling, defer to it. The required outcomes (a working DB, seed data that exercises the seven contexts, a way to swap IdP adapters in/out for local dev) survive regardless.

---

## 3.2 Prerequisites

| Tool | Version | Why |
|---|---|---|
| **Node.js** | LTS (≥ 20.x as of late 2025) | Backend + frontend runtime |
| **pnpm** | ≥ 9.x | Package manager + workspace orchestration (lean per [§1.9](01-stack.md)) |
| **Docker** (or Colima / Podman / Lima) | recent | Local Postgres + Testcontainers for integration tests |
| **Git** | recent | Version control |
| **psql** (Postgres CLI) | matches your DB version | Inspecting / querying local DB |

Optional but recommended:

- **direnv** or equivalent for per-project env var loading.
- **mise** / **asdf** / **fnm** / **nvm** for Node version pinning per the repo's `.nvmrc` or `.node-version`.

Verify:

```bash
node --version           # v20.x or later
pnpm --version           # 9.x or later
docker --version         # any recent
psql --version           # matches DB
```

---

## 3.3 First-time setup

From a fresh clone:

```bash
# 1. Clone the monorepo (path depends on monorepo location)
git clone <monorepo-url>
cd <monorepo-root>

# 2. Install dependencies (workspace-wide)
pnpm install

# 3. Copy env template; fill in local values
cp apps/bpp/.env.example apps/bpp/.env.local
cp apps/bpp-admin/.env.example apps/bpp-admin/.env.local
# Edit both .env.local files per §3.5

# 4. Start local Postgres
docker compose -f apps/bpp/docker-compose.dev.yml up -d
# OR start your monorepo's shared Postgres container if one exists

# 5. Run migrations against the local DB
pnpm --filter bpp db:migrate

# 6. Seed initial data (System Admin, sample Org + Store + Products, etc.)
pnpm --filter bpp db:seed

# 7. Verify by running tests
pnpm --filter bpp test:unit
pnpm --filter bpp test:integration   # requires Docker for Testcontainers
```

If all of the above complete green, you have a working dev environment.

---

## 3.4 Running locally

### Backend dev server

```bash
pnpm --filter bpp dev
# starts on http://localhost:3000 (or whatever PORT is set in .env.local)
```

The backend reloads on source changes (typically via `tsx --watch` or framework equivalent).

### Frontend dev server

```bash
pnpm --filter bpp-admin dev
# starts on http://localhost:5173 (Vite default)
```

Vite proxies API calls to the backend per `vite.config.ts`.

### Both together

```bash
# from monorepo root, using Turborepo (D9 lean) or equivalent:
pnpm turbo dev --filter bpp --filter bpp-admin
# OR run them in separate terminals
```

Open `http://localhost:5173` in a browser; sign in flow should redirect through the IdP sandbox (§3.7) and back.

### Watching outbox dispatcher / background jobs

By default the dev server starts the in-process outbox dispatcher + the scheduled-jobs runner (idempotency cleanup, session expiry sweep, etc.) on the same Node process. To run them in a sibling worker for closer-to-production behavior:

```bash
pnpm --filter bpp worker
# starts the outbox dispatcher + scheduled jobs in a separate process
```

---

## 3.5 Environment configuration

### Three layers (per [§6.5.1 of handoff](../handoff/06-operational.md))

| Layer | Where | Examples |
|---|---|---|
| **Compile-time** | Code (the capability catalog, feature flags) | Capabilities enumerated in `shared/auth/capability-catalog.ts` |
| **Deploy-time** | `.env.<env>` files + secrets manager | DB URL, IdP URL, CDS endpoint, signing key |
| **Runtime** | DB tables (System-Admin-edited) | Role → capability matrix; voucher content; operational toggles |

### `.env.local` file (backend)

```bash
# apps/bpp/.env.local — example contents (NEVER commit this file)

# Database
DATABASE_URL=postgresql://bpp:bpp@localhost:5432/bpp_dev

# IdP (per D3 — your chosen provider; see §3.7)
IDP_ISSUER_URL=https://your-idp-sandbox.example.com
IDP_CLIENT_ID=<dev-client-id>
IDP_CLIENT_SECRET=<dev-client-secret>
IDP_REDIRECT_URI=http://localhost:3000/auth/callback

# Session
SESSION_COOKIE_SECRET=<random-32-bytes-base64>
SESSION_COOKIE_DOMAIN=localhost
SESSION_TTL_DAYS=30

# Object storage (per D4 — your chosen provider; see §3.8 fallbacks)
OBJECT_STORAGE_ENDPOINT=http://localhost:9000      # MinIO if used locally
OBJECT_STORAGE_BUCKET=bpp-media-dev
OBJECT_STORAGE_ACCESS_KEY=<local-dev-key>
OBJECT_STORAGE_SECRET_KEY=<local-dev-secret>

# Email
EMAIL_PROVIDER=console                              # logs to console; no real send
EMAIL_FROM_ADDRESS=noreply@bpp-dev.local

# Beckn (Phase 6+; safe to leave unset until then)
BPP_ID=bpp-dev.local
BPP_URI=http://localhost:3000/beckn
BECKN_REGISTRY_URL=https://registry-staging.beckn.network
BECKN_SIGNING_KEY_PATH=/var/secrets/bpp-dev-signing-key.pem
CDS_ENDPOINT=https://cds-staging.beckn.network

# Observability (Phase 8+; optional in dev)
LOG_LEVEL=debug
OTEL_EXPORTER_OTLP_ENDPOINT=
```

### `.env.local` file (frontend)

```bash
# apps/bpp-admin/.env.local

VITE_API_BASE_URL=http://localhost:3000
VITE_PUBLIC_BPP_NAME=BPP Dev
```

### Rules

- **Never commit `.env.local`** — only commit `.env.example` with safe placeholders.
- **`.env.example` is the contract** — if you add an env var to code, update `.env.example` in the same PR.
- **Per-environment isolation** ([§6.5.3 of handoff](../handoff/06-operational.md)): production never accepts dev IdP signatures, dev never accepts production Beckn signatures, etc. Use distinct IdP credentials per env.
- **Secrets in production** come from the monorepo's secret-management layer (Vault, AWS Secrets Manager, etc.) — not from env files.

---

## 3.6 Database

### Starting local Postgres

A simple `docker-compose.dev.yml` per package, or a shared monorepo Postgres:

```yaml
# apps/bpp/docker-compose.dev.yml — example
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: bpp_dev
      POSTGRES_USER: bpp
      POSTGRES_PASSWORD: bpp
    ports:
      - "5432:5432"
    volumes:
      - bpp_postgres_data:/var/lib/postgresql/data

volumes:
  bpp_postgres_data:
```

```bash
docker compose -f apps/bpp/docker-compose.dev.yml up -d
docker compose -f apps/bpp/docker-compose.dev.yml down       # stop, preserve data
docker compose -f apps/bpp/docker-compose.dev.yml down -v    # stop + wipe
```

### Migrations (per [§4.11 of `04-conventions.md`](04-conventions.md))

Lean is Drizzle + `drizzle-kit`:

```bash
# Generate a new migration from schema diffs
pnpm --filter bpp db:generate -- --name add_voucher_minimum_cart

# Review the generated SQL in apps/bpp/migrations/ before committing.
# Edit if needed; commit.

# Apply pending migrations
pnpm --filter bpp db:migrate

# Roll back the last migration locally (only for dev — production is forward-only)
pnpm --filter bpp db:rollback
```

### Reset + reseed

```bash
# Wipe + recreate + re-migrate + reseed
pnpm --filter bpp db:reset
```

Under the hood this is approximately:

```bash
docker compose -f apps/bpp/docker-compose.dev.yml down -v
docker compose -f apps/bpp/docker-compose.dev.yml up -d
# wait for postgres ready
pnpm --filter bpp db:migrate
pnpm --filter bpp db:seed
```

Useful when you've changed a migration in dev and want a clean slate.

### Inspecting

```bash
psql -h localhost -U bpp bpp_dev
# common queries:
\dt                              # list tables
\d+ orders                       # describe orders table
SELECT * FROM users LIMIT 10;
SELECT event_name, COUNT(*) FROM audit_records GROUP BY event_name;
```

### DB roles for Audit (per [§6.8 of context-playbooks](06-context-playbooks/6.8-audit.md))

Local dev typically runs everything as the `bpp` superuser, but **integration tests must use the role-separated DB user** that the production deploy uses — otherwise the audit append-only invariant isn't actually exercised. The migration that creates the roles runs in local dev too; only the connection user differs.

---

## 3.7 IdP sandbox

### The configurable-provider problem

D3 (IdP provider) is **deferred to monorepo**. The BPP code talks to an OIDC-compatible IdP through `IdpClientPort` (per [§6.1 of context-playbooks](06-context-playbooks/6.1-identity.md)). Local dev needs a working IdP somehow — three options:

| Option | When to use |
|---|---|
| **Fake IdP adapter** (recommended for dev) | Day-to-day frontend / backend development. Skips real OIDC. Fast iteration. |
| **Real provider sandbox account** (Clerk dev, Supabase Auth dev project, etc.) | When testing real OIDC flows, claims handling, edge cases. Required before Phase 6 (Bridge). |
| **Local IdP container** (Keycloak / Dex / Ory Hydra) | When the monorepo standardizes on self-hosted IdP. Heavier setup. |

### Fake IdP adapter (recommended for dev)

Implement a `FakeIdpAdapter implements IdpClientPort` in `platform-infra/idp/` (gated behind `NODE_ENV !== 'production'`):

```ts
// platform-infra/idp/fake-idp-adapter.ts (illustration)
export class FakeIdpAdapter implements IdpClientPort {
  // Returns a sign-in URL that immediately calls our callback with a known token.
  buildAuthUrl(state: string) {
    return `${process.env.APP_URL}/auth/callback?code=fake-${state}&state=${state}`;
  }
  async exchangeCode(code: string) {
    const userId = code.replace(/^fake-/, '');                   // route to a known fake user
    return {
      idToken: 'fake-token',
      claims: this.fakeUsers[userId] ?? this.fakeUsers.default,
    };
  }
  private fakeUsers = {
    'system-admin':  { sub: 'fake-sa',  email: 'sa@bpp-dev.local',    email_verified: true,  name: 'System Admin' },
    'org-owner':     { sub: 'fake-oo',  email: 'oo@bpp-dev.local',    email_verified: true,  name: 'Org Owner' },
    'store-admin':   { sub: 'fake-sad', email: 'sad@bpp-dev.local',   email_verified: true,  name: 'Store Admin' },
    'default':       { sub: 'fake-u1', email: 'user1@bpp-dev.local', email_verified: true,  name: 'Test User' },
  };
}
```

Wired in the composition root by env flag:

```ts
container.register({
  idpClient: process.env.IDP_PROVIDER === 'fake'
    ? asClass(FakeIdpAdapter).singleton()
    : asClass(OidcIdpAdapter).singleton(),
});
```

In `.env.local`:

```bash
IDP_PROVIDER=fake
```

The admin UI's sign-in screen can show a "dev users" picker that POSTs to `/auth/sign-in?code=fake-system-admin&state=...` directly.

### Real IdP sandbox account (when D3 is locked)

When D3 lands, the monorepo will provide:
- A dev IdP application (separate from staging / prod).
- Client ID + secret in the secrets manager.
- A sandbox set of test users.

At that point, set `IDP_PROVIDER=oidc` (or remove the env var) and configure `IDP_*` envs per §3.5. The `OidcIdpAdapter` handles the rest.

### Rules

- **The fake adapter is dev-only.** Production builds must fail if `IDP_PROVIDER=fake`.
- **Fake users mirror real claim shape.** `sub`, `email`, `email_verified`, `name`, optional `picture`, `locale`. JIT provisioning (per [§6.1](06-context-playbooks/6.1-identity.md)) should produce the same `User` from a fake claim as from a real one.
- **Don't import the fake adapter from non-dev code paths.** Lint may help enforce.

---

## 3.8 Seed data

`pnpm --filter bpp db:seed` runs a deterministic script that produces a usable local dev state.

### What the seed produces

- **System Admins**: at least one, mapped to the `system-admin` fake user.
- **Capability matrix**: default grants per role per [§5.1 of handoff](../handoff/05-cross-cutting.md).
- **PlatformCategory taxonomy**: a small but real-shaped tree (Food / Beverages / Electronics / Clothing — enough to test categorization).
- **Sample Organization** (`Acme Co`), owned by the `org-owner` fake user.
- **Sample Store** (`Toko Acme`, currency `IDR`, status `Active`), assigned to the `store-admin` fake user.
- **Sample Products**: a Matrix-mode product (`T-Shirt` with Size + Color), a Flat-mode product (`Laptop`), with `LocalizedText` content in `id` and `en`.
- **Sample Vouchers**: `WELCOME10` (10% off, store-wide).
- **Empty Inventory**: StockLevels auto-created via the subscription; seed bumps starting quantities to ~50 each.

### Reset + reseed

```bash
pnpm --filter bpp db:reset      # wipe + reseed; see §3.6
pnpm --filter bpp db:seed       # reseed without wipe (idempotent)
```

### Rules

- **Seed scripts are idempotent.** Running `db:seed` twice produces the same state. Use `ON CONFLICT DO NOTHING` or equivalent upsert.
- **Seed data is committed to the repo.** It IS the test data. Make it readable.
- **Don't seed real PII.** Use clearly fake email addresses (`@bpp-dev.local`); names like "Test User"; addresses that are obviously placeholders.
- **Reseed before demos.** Drift accumulates as you click around the UI.

---

## 3.9 Common dev tasks

### Adding a new use case

Per [§4.3 of `04-conventions.md`](04-conventions.md):

1. **Schema first** in `packages/bpp-contracts/src/<context>/<use-case-name>.ts` — input + output Zod schemas.
2. **Domain logic** in `contexts/<context>/domain/` if any new entity behavior needed.
3. **Use case** in `contexts/<context>/application/use-cases/<use-case-name>.ts` using the `useCase({ ... })` envelope.
4. **Capability** added to `shared/auth/capability-catalog.ts` if needed.
5. **HTTP route** in `contexts/<context>/interfaces/http/` (or in `admin-api/` if first-party-only).
6. **Bridge handler** in `bridge/inbound/` if Beckn-driven.
7. **DI registration** in `composition-root.ts` if introducing new ports.
8. **Frontend** in `apps/bpp-admin/src/features/<context>/` if it has a UI surface.
9. **Tests**: domain tests, use-case tests with fakes, integration tests for the route, per [§7 of `07-testing.md`](07-testing.md).
10. **Event registry**: register any new events in `shared/domain-events/registry.ts`.
11. **PII catalog**: update `design/pii.md` if the use case touches PII fields.

### Adding a new entity

1. **Domain entity** in `contexts/<X>/domain/entities/`.
2. **Value objects** as needed.
3. **State machine** in `domain/state-machines/` if applicable.
4. **Schema declaration** in `contexts/<X>/infrastructure/schema/`.
5. **Repository port** + Drizzle adapter.
6. **Migration**: `pnpm --filter bpp db:generate -- --name add_<entity>_table`.
7. **DI registration** for the repository.
8. **Domain tests** for invariants + state transitions.

### Adding a new event

1. **Event type** in `contexts/<X>/domain/events/<event-name>.ts` using `defineEvent({ ... })`.
2. **Registry entry** in `shared/domain-events/registry.ts` — include `retention_category` (drives Audit sweep).
3. **PII fields tagged** in `design/pii.md` if event payload carries PII.
4. **Emit from use cases** as `events: [SomeEvent.from(...)]`.
5. **Bridge subscription** if Bridge needs to react (catalog publisher subscribes to many catalog events).
6. **Audit** picks it up automatically via broad subscription — verify with a test.

### Adding a new capability

1. **Add to catalog** in `shared/auth/capability-catalog.ts` — name follows `<resource>.<verb>` convention.
2. **Default grants** in the seed script.
3. **Use `requireCapability(name, scope)`** in every use case that needs it (at the top of the body, before any mutation).
4. **System Admin matrix UI** (Phase 8) will surface the new capability automatically.

### Running tests

```bash
# Unit tests (fast — domain + application with fakes)
pnpm --filter bpp test:unit

# Integration tests (slower — real DB via Testcontainers)
pnpm --filter bpp test:integration

# Watch mode during development
pnpm --filter bpp test:unit --watch

# Single file
pnpm --filter bpp test contexts/order/domain/entities/order.test.ts

# E2E Beckn flows (slowest — tagged separately)
pnpm --filter bpp test:e2e
```

### Generating a Drizzle migration

```bash
# After editing schema files in contexts/<X>/infrastructure/schema/:
pnpm --filter bpp db:generate -- --name <descriptive-name>

# Review the generated SQL in apps/bpp/migrations/.
# Edit if needed; commit alongside the schema change.
```

### Updating `bpp-contracts`

When you change a Zod schema in `packages/bpp-contracts/`, both `apps/bpp` and `apps/bpp-admin` get the change via workspace resolution. Re-typecheck:

```bash
pnpm typecheck
```

---

## 3.10 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `psql: connection refused` | Postgres container not running | `docker compose -f apps/bpp/docker-compose.dev.yml up -d` |
| `migration failed: relation already exists` | Migration was applied previously; schema diff is stale | `pnpm --filter bpp db:reset` to wipe and reapply |
| Sign-in callback returns 401 | `IDP_PROVIDER` env mismatch with adapter wiring; or `IDP_REDIRECT_URI` mismatch with IdP config | Verify `.env.local`; check `composition-root.ts` |
| `port 5432 already in use` | Another Postgres instance running | `lsof -i :5432`; stop the conflicting instance or change `POSTGRES_PORT` in compose file |
| Testcontainers can't start a container | Docker daemon not running or permission issue | Start Docker; verify with `docker ps` |
| `unique constraint violation` on user email at sign-in | Seed leftover or stale local data | `pnpm --filter bpp db:reset` |
| Frontend can't reach backend | Backend not running on expected port, OR CORS config issue | Check both `.env.local` files; verify `VITE_API_BASE_URL` |
| `Idempotency-Key already used with different payload` error | Browser auto-resubmitted with a stale UUID | Refresh the page; clean browser state |
| Outbox events not delivering | Worker process not running (if running backend + worker split) | `pnpm --filter bpp worker` in a separate terminal |
| `bridge_inbox` dedup rejects test inbound payload | Replayed test with same `(transaction_id, message_id)` | Use a fresh transaction_id per test; or `DELETE FROM bridge_inbox` in test setup |
| `audit_subscriber` role can't UPDATE in a test | Working as intended — append-only invariant | If the test is wrong, use the right role; if the test expected mutability, the test is wrong |

---

## 3.11 What this section doesn't cover

| For | Read |
|---|---|
| Why the system is shaped the way it is | [handoff/](../handoff/README.md) |
| What to build | [`05-phases.md`](05-phases.md), [`06-context-playbooks/`](06-context-playbooks/README.md) |
| How to write code | [`04-conventions.md`](04-conventions.md) |
| How to test code | [`07-testing.md`](07-testing.md) |
| Stack rationale | [`01-stack.md`](01-stack.md) |
| Folder structure | [`02-repo-layout.md`](02-repo-layout.md) |
| Deployment | `08-deployment.md` (pending) |
| Operational runbooks | `09-runbooks.md` (pending) |

---

> **Next**: `09-runbooks.md` — operational tasks (signing-key rotation, retention sweep, stuck-events triage, etc.).
