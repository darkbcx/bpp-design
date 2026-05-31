# 8. Deployment

How to deploy the BPP package to non-local environments. This section is structured for a **standalone v1 deploy** — the scenario where the BPP ships independently before the host monorepo merger. When the monorepo arrives, its deployment story supersedes the specifics here; the **Required** items in each section survive regardless.

| Section | Topic |
|---|---|
| 8.1 | Reading this section |
| 8.2 | Deployment shape |
| 8.3 | Environment isolation |
| 8.4 | Secrets management |
| 8.5 | Backend deployment |
| 8.6 | Database deployment |
| 8.7 | Frontend deployment |
| 8.8 | CI/CD pipeline |
| 8.9 | First-time environment bootstrap |
| 8.10 | Beckn-specific deploy concerns |
| 8.11 | Post-deploy smoke tests |
| 8.12 | Migration to host monorepo |
| 8.13 | What this section doesn't cover |

---

## 8.1 Reading this section

D6 (backend hosting), D7 (DB hosting), and D8 (frontend hosting) are **deferred to the host monorepo**. This section assumes a **standalone v1** scenario where the BPP ships first; concrete tooling uses the inline leans from [`01-stack.md`](01-stack.md) (Fly.io / Neon / Cloudflare Pages) **as one reference model**, not commitments.

Two layers:

- **Required:** architecture-imposed deploy properties (data residency, env isolation, secrets handling, forward-only migrations, etc.). These survive any hosting choice.
- **Lean (v1 standalone):** concrete tooling for the reference model. Substitute when the monorepo lands.

If you're reading this after the monorepo has merged, treat this section as historical — defer to the monorepo's deploy docs for current procedures.

---

## 8.2 Deployment shape

### Required

The BPP comprises **three deployable artifacts** (per [§2.3 of `02-repo-layout.md`](02-repo-layout.md)):

| Artifact | What it does | Where it runs |
|---|---|---|
| **Backend service** (`apps/bpp/`) | HTTP server (admin-api + Bridge inbound handlers), outbox dispatcher, scheduled jobs | A long-running compute host with outbound network access (to IdP, CDS, registry, object storage) |
| **Frontend SPA** (`apps/bpp-admin/`) | Static React bundle | Any static-asset host with HTTPS + cache control |
| **Database** | Postgres-compatible RDB with role separation for Audit | Managed Postgres OR self-managed |

Plus **operational dependencies**:

- **Object storage** (D4) — signed-URL provider for catalog media.
- **Email service** — for invitations.
- **Beckn registry** (Phase 6+) — for participant verification + key lookup.
- **CDS endpoint** (Phase 6+) — for catalog publishing.
- **IdP** (D3) — OIDC provider.
- **Secrets manager** — for credentials.
- **Observability backend** (D5) — for logs / metrics / traces.

The backend can run as a **single process** (HTTP + dispatcher + jobs all in one) or as **separate processes** (web / worker split). Both are supported by the architecture's topology-neutrality ([§5.2.8 of handoff](../handoff/05-cross-cutting.md)).

### Lean (v1 standalone)

```
┌──────────────────┐         ┌──────────────────────┐
│   Cloudflare     │         │       Fly.io         │
│   Pages          │ ──────▶ │       Backend        │
│   (Admin UI)     │  HTTPS  │   (web + worker      │
└──────────────────┘         │   processes)         │
                             └──────────┬───────────┘
                                        │
                                        ├─▶ Neon (Postgres)         [Singapore region]
                                        ├─▶ Cloudflare R2 (media)
                                        ├─▶ <IdP per D3>             (e.g., Clerk)
                                        ├─▶ Email provider
                                        ├─▶ Beckn registry           (Phase 6+)
                                        ├─▶ CDS endpoint             (Phase 6+)
                                        └─▶ Grafana Cloud (OTel)     (Phase 8+)
```

---

## 8.3 Environment isolation

### Required

Per [§6.5.3 of handoff](../handoff/06-operational.md):

- **Distinct environments** for dev / staging / prod.
- **Distinct credentials per environment** for IdP, CDS, Beckn registry, object storage, observability.
- **Production never accepts test-issued Beckn signatures.** Staging and prod use different Beckn networks (staging registry vs production registry).
- **Production never connects to dev/staging DBs.** Cross-environment DB access is forbidden.
- **Each environment has its own `bpp-id` and `bpp-uri`** — the BPP appears on different Beckn networks per env.
- **No PII flows from prod to non-prod.** Don't restore prod backups to staging without scrubbing.

### Recommended environment matrix

| Aspect | dev (per-engineer local) | staging | prod |
|---|---|---|---|
| Postgres | Docker container | Neon / managed | Neon / managed |
| Backend host | localhost | Fly.io app: `bpp-staging` | Fly.io app: `bpp-prod` |
| Frontend host | Vite dev server | Cloudflare Pages preview branch | Cloudflare Pages production branch |
| IdP | Fake adapter (per [§3.7 of `03-dev-setup.md`](03-dev-setup.md)) | Real IdP staging app | Real IdP production app |
| Beckn network | None | Beckn staging | Beckn production |
| `bpp-id` | n/a | `bpp-staging.example.com` | `bpp.example.com` |
| Signing key | n/a | Staging key in secrets | Production key in secrets |
| CDS endpoint | n/a | CDS staging | CDS production |
| Observability | console | Grafana Cloud staging | Grafana Cloud production |
| Data residency | local | Indonesia-region (or nearest) | **Indonesia-resident** (PDP) |

### Branch ↔ environment mapping (lean)

| Branch | Auto-deploys to |
|---|---|
| feature branches | Cloudflare Pages preview URLs (frontend only); ephemeral backend if budget permits |
| `main` | Staging |
| Tagged release (e.g., `v1.2.3`) | Production (with manual approval gate) |

Production deploys are **never auto-deployed from main** — always gated on:
1. Manual approval by an authorized engineer.
2. Green CI.
3. Released-from a tagged commit (audit trail).

---

## 8.4 Secrets management

### Required

- **Never in code.**
- **Never in env-checked-in files.** `.env.example` only.
- **Stored in a secrets manager** with per-environment isolation and access audit.
- **System Admin seed credentials** distributed via secure operational channel — never recoverable in-band.
- **Beckn signing keys** are **ONIX's responsibility** post-[ADR-0022](../decisions/0022-onix-protocol-gateway.md); not in BPP secrets. Rotation is a vendor operation (see [§9.5 of `09-runbooks.md`](09-runbooks.md)).
- **No secret reuse across environments** (a leaked staging secret cannot grant prod access).

### Lean

| Tool | Where it shines |
|---|---|
| **Fly.io secrets** (lean if backend on Fly) | First-class for backend; per-app namespace |
| **Cloudflare secrets** (lean if frontend on Cloudflare) | For frontend build-time secrets |
| **1Password / Doppler / Bitwarden** | Team coordination + access audit |
| **HashiCorp Vault / AWS Secrets Manager / GCP Secret Manager** | When the monorepo standardizes |

Secrets the BPP needs per environment:

- `DATABASE_URL` (with restricted user; not the DB owner)
- `SESSION_COOKIE_SECRET` (random 32 bytes per env; never reuse)
- IdP: `IDP_CLIENT_SECRET`
- Object storage: access key / secret key
- Email: provider API key
- Observability: OTel exporter credentials
- **(Optional, depends on ADR-0022 §10 N1 resolution)** BPP CounterSignature signing key — only if N1 = (a) — narrow key for synchronous Ack/Nack responses only. If N1 = (b), the BPP holds no signing keys.

**Beckn signing keys and registry credentials are NOT BPP secrets** post-[ADR-0022](../decisions/0022-onix-protocol-gateway.md). They live in ONIX's vendor-managed secrets layer.

**ONIX-related BPP config** (not secrets — just URLs):
- `BPP_ID` — set in outbound `context.bpp_id`.
- `BPP_URI` — set in outbound `context.bpp_uri`; value equals the ONIX network-facing URL.
- `ONIX_ENDPOINT` — the BPP-facing URL of ONIX (private; reachable only from the BPP).

### Audit DB role separation in production

Per [§6.8 of context-playbooks](06-context-playbooks/6.8-audit.md), Audit uses four DB roles. In production, each role must have **distinct credentials** stored in the secrets manager:

- `audit_subscriber` — used by the dispatcher process for INSERTs.
- `audit_pii_scrubber` — used by `ScrubUser` orchestration only.
- `audit_cleanup` — used by the retention sweep job.
- `audit_reader` — used by read-side use cases.

Application code receives the correct credential per use case via DI registration — see the per-environment `DATABASE_URL_*` env vars.

---

## 8.5 Backend deployment

### Required

- **Long-running compute** (not edge / serverless for v1 — the outbox dispatcher needs persistent state).
- **Outbound network access** to IdP, **ONIX** (per [ADR-0022](../decisions/0022-onix-protocol-gateway.md)), object storage, observability, email. No direct connection to Beckn registry or CDS — ONIX handles those.
- **Process supervision** — auto-restart on crash.
- **Graceful shutdown** — drain in-flight requests; flush outbox; close DB connections.
- **Health checks**: `/healthz` (liveness) + `/readyz` (readiness — confirms DB, IdP discovery doc, **ONIX reachability**).
- **Forward-only migrations** apply automatically on deploy (or via a manual gate, but never roll back in production).

### Lean: Fly.io

```toml
# apps/bpp/fly.toml (illustration)
app = "bpp-prod"
primary_region = "sin"   # Singapore — nearest available to Indonesia in standard Fly.io regions

[build]
  dockerfile = "Dockerfile"

[env]
  PORT = "8080"
  LOG_LEVEL = "info"
  # Non-secret env; secrets via `fly secrets set`

[[services]]
  internal_port = 8080
  protocol = "tcp"

  [services.concurrency]
    type = "connections"
    hard_limit = 100
    soft_limit = 80

  [[services.tcp_checks]]
    interval = "10s"
    timeout = "2s"

  [[services.http_checks]]
    interval = "30s"
    timeout = "5s"
    method = "GET"
    path = "/healthz"

[[mounts]]
  source = "bpp_data"
  destination = "/data"
```

### web + worker split (recommended for production)

Two Fly.io processes:

```toml
[processes]
  web = "node dist/web/main.js"
  worker = "node dist/worker/main.js"
```

- `web` handles HTTP (admin-api + Bridge inbound).
- `worker` runs the outbox dispatcher + scheduled jobs (retention sweep, session expiry, quote expiry, etc.).

### Indonesia region

Fly.io's nearest region to Indonesia is **Singapore (`sin`)**. For Indonesia-resident DB (required for PDP), the DB must be hosted in an Indonesia region (see §8.6). The backend in Singapore + DB in Jakarta is acceptable (low latency over private peering); confirm during planning.

Alternatives if Indonesia-region backend is required: AWS `ap-southeast-3` (Jakarta) on ECS / EKS; GCP `asia-southeast2` (Jakarta) on Cloud Run / GKE.

### Deploy

```bash
# From CI (preferred) — see §8.8
fly deploy --app bpp-prod --image-label v1.2.3 --strategy rolling

# Or manually (for emergency hotfix)
fly deploy --app bpp-prod
```

`--strategy rolling` ensures zero-downtime — Fly.io spins up new instances, waits for health checks, then drains the old ones.

---

## 8.6 Database deployment

### Required

- **ACID-compliant Postgres-equivalent** (per [§1.2 of `01-stack.md`](01-stack.md)).
- **Indonesia-resident storage** for production ([§5.6.10 of handoff](../handoff/05-cross-cutting.md)).
- **Encryption at rest** (DB-level).
- **TLS for all client connections.**
- **Automated backups** with documented RPO / RTO.
- **Point-in-time recovery** (PITR) for production.
- **Connection pooling** between app and DB (PgBouncer or provider-native).
- **DB roles configured** per [§8.4 audit role separation](#84-secrets-management).

### Lean: Neon (production) / Neon free tier (staging) / Docker container (dev)

| Env | Lean |
|---|---|
| dev | Docker Postgres 16 (per [§3.6 of `03-dev-setup.md`](03-dev-setup.md)) |
| staging | Neon project (Singapore region, free tier) — preview branches per PR |
| prod | Neon project (Singapore or Jakarta region — confirm with Neon for IDN residency) |

Alternatives that satisfy the Required list:

- **Supabase Postgres** (if D3 = Supabase Auth — bundles)
- **AWS RDS** in `ap-southeast-3` (Jakarta) — guaranteed IDN residency
- **GCP Cloud SQL** in `asia-southeast2` (Jakarta) — same
- **Self-managed Postgres** on a Jakarta-region VM (more ops, full control)

### Migrations on deploy

Lean approach with Drizzle:

1. CI runs `drizzle-kit generate` from schema changes during PR review (per [§3.6 of `03-dev-setup.md`](03-dev-setup.md)). Generated SQL committed.
2. On deploy: CI runs `pnpm db:migrate` against the target env BEFORE rolling out new app instances.
3. If migration fails, deploy aborts. Manual intervention required.

This works because BPP migrations are **forward-compatible during the window** (per [§4.11 of `04-conventions.md`](04-conventions.md) — Expand → Backfill → Contract for breaking changes).

```bash
# Lean CI step
DATABASE_URL=$PROD_DB_URL pnpm --filter bpp db:migrate
```

### DB role setup

The Audit DB roles are created by a migration; binding them to actual credentials is operational. Sketch:

```sql
-- migration creates the roles (per §6.8 of context-playbooks)
CREATE ROLE audit_subscriber NOLOGIN;
-- ... etc.

-- operational step (per env): bind to actual users with secrets-manager-provided passwords
CREATE USER bpp_audit_subscriber_prod WITH PASSWORD '<from-secrets-manager>';
GRANT audit_subscriber TO bpp_audit_subscriber_prod;
-- ... etc.
```

The app's DI registration uses `DATABASE_URL_AUDIT_SUBSCRIBER`, `DATABASE_URL_AUDIT_READER`, etc. — distinct connection strings per role.

---

## 8.7 Frontend deployment

### Required

- **Static SPA hosting** with HTTPS and cache control headers.
- **CORS configured** to permit the BPP backend origin.
- **Build-time env injection** for non-secret values (`VITE_API_BASE_URL`, etc.).
- **No secrets in the bundle.** Frontend never carries `IDP_CLIENT_SECRET` or `DATABASE_URL`.
- **Per-environment build** — staging and prod produce different bundles.

### Lean: Cloudflare Pages

```toml
# apps/bpp-admin/wrangler.toml (illustration; depends on tooling)
name = "bpp-admin-prod"
compatibility_date = "2025-01-01"

[[env.production]]
build_command = "pnpm --filter bpp-admin build"
output_directory = "dist"

[[env.production.vars]]
VITE_API_BASE_URL = "https://api.bpp.example.com"
```

Alternative: **Vercel** for similar SPA hosting with edge caching and per-PR preview URLs.

Connect the GitHub repo to Cloudflare Pages or Vercel; auto-deploy:

- Feature branch → preview URL (e.g., `pr-42.bpp-admin-staging.pages.dev`)
- `main` → staging
- Tag → production (with approval gate)

### Frontend env vars

Only **non-secret** values prefixed with `VITE_`:

```bash
# .env.production
VITE_API_BASE_URL=https://api.bpp.example.com
VITE_PUBLIC_BPP_NAME=Acme BPP
```

Secrets stay in the backend; the frontend calls the backend, which uses secrets server-side.

---

## 8.8 CI/CD pipeline

### Required

- **PR pipeline**: typecheck, lint (incl. boundary rules per [§2.8 of `02-repo-layout.md`](02-repo-layout.md)), unit tests, integration tests, build. **Merge gated on green.**
- **Staging deploy pipeline**: runs on merge to `main` (or equivalent). Deploys backend + frontend + runs migrations.
- **Production deploy pipeline**: runs on tag push, with **manual approval gate**.
- **Rollback procedure** documented and rehearsed (see [§9.10](09-runbooks.md)).

### Lean: GitHub Actions

```yaml
# .github/workflows/ci.yml (sketch)
name: CI
on:
  pull_request:
    branches: [main]
jobs:
  ci:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: ci }
        ports: ["5432:5432"]
        options: --health-cmd "pg_isready -U postgres"
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm --filter bpp test:unit
      - run: pnpm --filter bpp test:integration
        env: { DATABASE_URL: postgresql://postgres:ci@localhost:5432/postgres }
      - run: pnpm --filter bpp-admin test
      - run: pnpm build
```

```yaml
# .github/workflows/deploy-staging.yml (sketch)
name: Deploy staging
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: pnpm install --frozen-lockfile
      - name: Run migrations
        run: pnpm --filter bpp db:migrate
        env: { DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }} }
      - name: Deploy backend
        run: flyctl deploy --app bpp-staging --strategy rolling
        env: { FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} }
      # frontend deploys via Cloudflare Pages auto-build on push
```

```yaml
# .github/workflows/deploy-production.yml (sketch)
name: Deploy production
on:
  push:
    tags: ['v*.*.*']
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production   # GitHub Environment with required reviewer
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: pnpm install --frozen-lockfile
      - name: Run migrations
        run: pnpm --filter bpp db:migrate
        env: { DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }} }
      - name: Deploy backend
        run: flyctl deploy --app bpp-prod --image-label ${{ github.ref_name }} --strategy rolling
        env: { FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} }
```

### Rules

- **CI uses ephemeral Postgres** for integration tests (services container above) — fast, isolated, no leakage.
- **Migrations run before app deploy.** If migration fails, no new app version starts.
- **Manual approval for production** via GitHub Environment protection rules.
- **Tags are immutable.** Never delete a release tag.
- **Builds are reproducible** — pinned versions in `pnpm-lock.yaml`; Docker base image pinned.

---

## 8.9 First-time environment bootstrap

When standing up a fresh environment (e.g., new staging from scratch):

### Procedure

1. **Provision DB**:
   - Create the Neon project (or equivalent) in the chosen region.
   - Create the application user with restricted permissions (not the DB owner).
   - Record `DATABASE_URL` in the secrets manager.
2. **Create object storage bucket**:
   - Provision R2 bucket (or equivalent).
   - Create access credentials with bucket-scoped permissions.
   - Record in secrets.
3. **Configure IdP**:
   - Create a new OIDC application at the IdP provider for this environment.
   - Set callback URL: `https://<bpp-host>/auth/callback`.
   - Record client ID + secret in secrets.
4. **Generate session cookie secret**:
   - 32 random bytes, base64-encoded.
   - Per-environment; never reuse across envs.
5. **(Phase 6+) Beckn key material**:
   - Generate signing key pair (per Beckn-prescribed algorithm).
   - Record private key in secrets; register public key with the appropriate Beckn registry (staging or prod).
   - Set `BPP_ID` and `BPP_URI` for the env.
6. **Configure email provider** with per-env API key.
7. **Set up observability** (D5):
   - Create Grafana Cloud stack (or equivalent) for the env.
   - Configure OTel exporter credentials in secrets.
8. **Create Fly.io app(s)**:
   - `flyctl apps create bpp-staging` (and `bpp-prod` separately).
   - Set all secrets: `flyctl secrets set --app bpp-staging DATABASE_URL=... ...`.
9. **Create Cloudflare Pages project** for the frontend env.
10. **Run initial deploy** via CI:
    - First push triggers migration + app deploy.
    - Verify health checks.
11. **Seed initial data**:
    - At minimum: System Admin user(s) — `external_subject_id` must match the IdP `sub` of the operator.
    - Capability matrix grants.
    - PlatformCategory taxonomy.
    - Run the documented seed script: `DATABASE_URL=... pnpm --filter bpp db:seed:initial`.
12. **Smoke test** per [§8.11](#811-post-deploy-smoke-tests).
13. **Document the environment** in the operations log: who created it, when, why, where the credentials live.

---

## 8.10 Beckn-specific deploy concerns (BPP + ONIX)

Per [ADR-0022](../decisions/0022-onix-protocol-gateway.md), the BPP integrates with the Beckn network through **ONIX** — a vendor-provided binary deployed per-BPP. ONIX holds the signing key, registry credentials, and CDS endpoint config. The BPP holds only `BPP_ID`, `BPP_URI` (= the ONIX network-facing URL), and `ONIX_ENDPOINT` (the BPP-facing URL of ONIX).

### Required

- **`bpp-id` per environment.** Each environment is a distinct Beckn participant — staging on staging registry, prod on prod registry.
- **One ONIX instance per BPP environment.** Each deployment env has its own ONIX with its own signing key + registry credentials.
- **`BPP_URI` equals the ONIX network-facing URL.** The network sees ONIX as the BPP endpoint.
- **`ONIX_ENDPOINT` (BPP-facing) is private.** Network-level isolation is the security model — no auth between BPP and ONIX.
- **Registry registration is an ONIX vendor operation**, not BPP code. The BPP never holds registry credentials.
- **No staging traffic on production network.** Strictly enforced — separate ONIX instances per env, signing with separate keys.

### Registration procedure (Phase 6+) — ONIX-side

When deploying to a Beckn-connected environment for the first time:

1. **Provision the ONIX instance** for the environment (vendor procedure).
2. **Configure ONIX**:
   - `bpp-id`: e.g., `bpp-staging.example.com`
   - `bpp-uri`: e.g., `https://onix-staging.bpp.example.com` (ONIX's network-facing URL)
   - Network domain (e.g., `retail`, `mobility`)
   - Signing key material + registry credentials
   - The BPP-facing endpoint URL (private, reachable only from the BPP)
3. ONIX submits registration to the appropriate Beckn registry.
4. **Configure the BPP** to point at ONIX: set `BPP_ID`, `BPP_URI` (= ONIX network-facing URL), `ONIX_ENDPOINT` (= ONIX BPP-facing URL).
5. Test inbound + outbound with a stub BAP via the ONIX path.
6. Record registration details in the operations log (vendor-side record).

### Updating registration

If `bpp-uri` changes (ONIX host migration) OR signing key rotates (per [§9.5 of `09-runbooks.md`](09-runbooks.md)) — vendor-side operation against ONIX; the BPP's only update is potentially `BPP_URI` if the ONIX host moves.

### Phase-aware deploy

In **Phase 1–5** (before Phase 6 Bridge lands), the BPP can deploy to staging / prod without ONIX integration — Beckn-related env vars (`BPP_ID`, `BPP_URI`, `ONIX_ENDPOINT`) stay unset. The Bridge code paths short-circuit if these are empty.

When **Phase 6 lands**, the ONIX instance must be provisioned and registered as a coordinated step alongside the deploy of the Bridge code. Don't deploy Bridge code to production without ONIX in place and registered.

### ONIX zero-downtime restarts

ONIX supports zero-downtime restarts (vendor-provided). During an ONIX restart, BPP outbound POSTs get transient 5xx responses; `bridge_outbox` retries with exponential backoff (matching ION-9001 retry semantics). No special "drain BPP" pattern is needed — inbox absorbs the transient errors.

---

## 8.11 Post-deploy smoke tests

After every production deploy:

| Check | What to verify | How |
|---|---|---|
| **Health** | App is up | `curl https://api.bpp.example.com/healthz` returns 200 |
| **Readiness** | DB + IdP + ONIX reachable | `curl https://api.bpp.example.com/readyz` returns 200 |
| **Frontend** | Static bundle served | Open admin URL in browser; sign-in page renders |
| **Sign-in flow** | OIDC round-trip works | Sign in as a known test user; cookie set; `/me` returns user |
| **Migration** | Schema is current | Query `_drizzle_migrations` table (or equivalent); latest migration is the deployed one |
| **Outbox dispatcher** | Worker process running | Create a test event in non-prod; verify it's dispatched within seconds; in prod, monitor `outbox.dispatcher.lag` |
| **Audit ingestion** | Audit subscriber active | Verify recent `audit_records` for the deploy's startup events |
| **ONIX reachability** (Phase 6+) | BPP can POST to `ONIX_ENDPOINT` | `curl <onix-endpoint>/health` (or vendor-defined) returns 200 |
| **Bridge inbound** (Phase 6+) | Inbound re-verification healthy | Send a known-good payload via ONIX from stub BAP; verify accepted with Ack + CounterSignature |
| **CDS publish** (Phase 6+) | Recent publish on a known-Active store reaches ONIX | Check `bridge_outbox` for recent `state: 'sent'` rows |
| **Page-worthy alerts** | None firing | Check observability dashboard |

If any check fails, **rollback per the runbook** ([§9.10 of `09-runbooks.md`](09-runbooks.md)).

---

## 8.12 Migration to host monorepo

When the host monorepo merger lands:

### What changes

- **Deployment story moves to the monorepo.** This section becomes historical.
- **The BPP package moves** from a standalone repo into `apps/bpp/`, `apps/bpp-admin/`, `packages/bpp-contracts/` within the monorepo (per [§2.10 of `02-repo-layout.md`](02-repo-layout.md)).
- **CI/CD adopts the monorepo's pipelines.** GitHub Actions workflows in the BPP repo become monorepo workflows; `pnpm --filter bpp ...` patterns translate naturally.
- **Secrets management adopts the monorepo's tool** (Vault, Doppler, etc.).
- **Hosting may change** — D6/D7/D8 leans get replaced by whatever the monorepo standardizes on.
- **The Bridge stays isolatable** (per [§2.7 of CLAUDE.md](../CLAUDE.md), [§6.3 of handoff](../handoff/06-operational.md)) — even if the rest of the BPP merges, the Bridge can move to a separate deployable later.

### What doesn't change

The **Required** items in each section above survive:

- ACID Postgres with role separation for Audit.
- Indonesia-resident DB.
- Per-environment isolation with distinct credentials.
- Secrets never in code.
- Forward-only migrations as reviewable artifacts.
- `bpp-id` per environment.
- Production never accepts test-issued signatures.
- Web + worker process model (or single-process with the same code paths).

### Migration checklist

When merging:

1. Move the three packages into the monorepo's structure (preserve git history if possible).
2. Update `package.json` workspace references.
3. Re-run lint with the monorepo's boundary-rule configuration; adjust paths.
4. Adopt the monorepo's CI patterns; port the BPP-specific test stages.
5. Migrate secrets to the monorepo's secrets management.
6. Update deploy targets per the monorepo's tooling.
7. Cut over DNS / Fly.io apps to the monorepo's deploy outputs at a controlled cutover window.
8. Decommission the old standalone deploy after parallel-run verification.

---

## 8.13 What this section doesn't cover

| For | Read |
|---|---|
| Why the system is shaped the way it is | [handoff/](../handoff/README.md) |
| Local development setup | [`03-dev-setup.md`](03-dev-setup.md) |
| Operational procedures (rotation, failover, incident response) | [`09-runbooks.md`](09-runbooks.md) |
| Testing strategy in CI | [`07-testing.md`](07-testing.md) |
| What the host monorepo's deploy story should look like | (Belongs in the monorepo's own docs when it lands) |
| Disaster recovery drills | Pre-launch readiness; [`09-runbooks.md`](09-runbooks.md) Backup / Restore runbooks |
| Performance tuning | Post-launch; capacity planning per real traffic |

---

> **End of developer-guide.** From here, the implementing team is the source of truth.
