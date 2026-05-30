# 6. Operational stance

This section is the **operations-facing companion** to the architecture. The previous sections defined the domain, the contexts, and the cross-cutting patterns. This one covers how to **build, deploy, observe, and run** the system without violating any of them.

The charter explicitly defers most operational decisions ([§2.9](02-principles.md) — "deployment topology is deferred"). What follows is the **non-negotiable operational shape** the architecture requires, plus reasonable v1 defaults where guidance helps. Anything not addressed here is the implementing team's call.

| # | Topic |
|---|---|
| 6.1 | Testing strategy |
| 6.2 | Observability |
| 6.3 | Deployment topology |
| 6.4 | External dependencies |
| 6.5 | Configuration and secrets |
| 6.6 | Rate limiting and resource constraints |
| 6.7 | Operational runbooks (what the team will need) |

---

## 6.1 Testing strategy

Testing follows the layered architecture ([§2.4](02-principles.md)). Each layer has its own posture; the Bridge has the most distinctive one.

> The presence of Beckn payloads in any test outside the Bridge's own test suite is a **red flag worth investigating** ([§4](03-beckn-integration.md), per the charter).

### 6.1.1 Domain Layer — pure unit tests

- No I/O. No fixtures. No frameworks beyond the test runner.
- Entities, value objects, state transitions, invariants — exercised by direct construction and method calls.
- If a domain test needs a Beckn payload to make sense, the test is wrong.

### 6.1.2 Application Layer — use-case tests with fakes

- Ports are mocked/faked.
- Each use case is tested for:
  - **Happy path** — preconditions met, expected state change, expected events emitted.
  - **Authorization** — `requireCapability` is called before any mutation (per [§5.1](05-cross-cutting.md)).
  - **Idempotency** — repeat with same key returns cached result; different payload returns typed error (per [§5.4](05-cross-cutting.md)).
  - **Compensation** — multi-context orchestrations release Reservations / revert vouchers on failure (per [§5.3](05-cross-cutting.md)).
  - **State preconditions** — e.g., `Order.Confirm` rejects when status ≠ `Initiated`.
- No Beckn payloads, no transport, no real DB.

### 6.1.3 Infrastructure Layer — adapter integration tests

Test each adapter against the real external system (or a faithful test double):

| Adapter | Tested against |
|---|---|
| DB / persistence | Real database (testcontainer) |
| Outbox dispatcher | Real DB + real message bus (or in-process queue, in monolith) |
| IdP adapter | IdP test instance or contract tests against IdP fixtures |
| Email adapter | Email service sandbox / capture inbox |
| Registry adapter | Network-stubbed registry; real registry sandbox in staging |

Adapters are interchangeable per the Dependency Rule ([§2.3](02-principles.md)); swapping (e.g., Postgres → another DB engine) must not break the inner layers.

### 6.1.4 Interface Layer

**Beckn Bridge** — its own posture, with the heaviest test budget:

- **Inbound parsing**: every Beckn message type, every supported protocol version, tested against real payloads from `ion-specs` examples. Signature verification, schema validation, mapping to Application-Layer calls.
- **Outbound projection**: domain state → Beckn message, per protocol version. Verify mapping registry coverage.
- **Asynchronous correlation**: request → eventual callback, tied by `transaction_id` / `message_id`.
- **Idempotency at the protocol boundary**: duplicate inbound message recognized and short-circuited.
- **Version negotiation**: multiple protocol versions coexist; each routed to the correct mapper.
- **Error mapping**: every domain error has a Beckn-error projection; unmappable cases produce generic protocol errors *and* log the mismatch (per [§4 in charter](03-beckn-integration.md)).
- **CDS publishing**: every event the Bridge subscribes to produces the expected `catalog/publish` payload (per [§4.6](04-bounded-contexts/4.6-order.md) and [§3](03-beckn-integration.md)).

**Admin UI / First-party APIs**:
- Standard contract tests against the Application Layer.
- E2E tests for primary admin journeys (Org creation → Store creation → product publish → catalog activation → republish).

### 6.1.5 End-to-end Beckn flows

Run as integration tests with a **stubbed BAP counterpart**:

- `/search` (CDS) → resource visibility per store state
- `/select` → quote with correct snapshots
- `/init` → reservations held, contact_snapshot populated
- `/confirm` → reservations converted, voucher recorded, callbacks fired
- `/cancel` → reservations released or voucher reverted depending on state
- `/status` → correct state projection

These exist to verify the Bridge ↔ Order interface holistically, not to test the domain (which is already tested in 6.1.1–6.1.2).

### 6.1.6 Property tests for state machines

State machines worth exercising with property-based testing:
- Order state machine ([§4.6](04-bounded-contexts/4.6-order.md))
- Store lifecycle ([§4.2](04-bounded-contexts/4.2-tenancy.md))
- Reservation lifecycle ([§4.4](04-bounded-contexts/4.4-inventory.md))
- Invitation lifecycle ([§4.2](04-bounded-contexts/4.2-tenancy.md))

Property: from any reachable state, only the declared transitions are accepted; all others are rejected with typed errors.

### 6.1.7 What we deliberately do not test

- Generated boilerplate (DTOs, ORM declarations).
- IdP internal mechanics (we trust the IdP).
- Specific cosmetic UI details (covered by visual regression separately, if at all).
- ION/Beckn-internal correctness — we trust `ion-specs`.

---

## 6.2 Observability

The system's observability has three planes:

| Plane | What it tracks |
|---|---|
| **Logs** | Operational events (errors, warnings, info) — for debugging |
| **Metrics** | Numerical telemetry (counts, latency, queue depth) — for trending and alerting |
| **Traces** | Request/event flows across contexts — for understanding causality |

Audit ([§4.7](04-bounded-contexts/4.7-audit.md)) is a fourth, **business** plane — not operational. Don't conflate them.

### 6.2.1 Structured logging

- **Format**: structured (JSON) at the wire; human-rendered locally.
- **Standard fields**: `timestamp`, `level`, `service`, `context_name`, `correlation_id`, `causation_id` where applicable, `user_id` (if known and authorized to log), `active_org_id`, `active_store_id`.
- **PII discipline** ([§5.6](05-cross-cutting.md)): every logger goes through the redacting layer. Raw email, names, IPs, free-text user input never reach logs in plain.
- **Log levels**: `error` for unexpected; `warn` for degraded-but-handled; `info` for state transitions and significant events; `debug` for development only.

### 6.2.2 Metrics

Recommended metric families:

| Family | Examples |
|---|---|
| Use-case throughput | `usecase.invoke{name=...} count, latency` |
| Authorization | `auth.denied{capability=...} count` |
| Outbox | `outbox.pending count, dispatcher.lag` |
| Inbox / subscribers | `subscriber.events_processed{name=...}`, `subscriber.dlq_count` |
| Stuck events | `stuck_events count{subscription=...}` |
| Bridge inbound | `beckn.inbound{action=...} count, latency, error count, signature_failure count` |
| Bridge outbound | `beckn.outbound{action=...} count, dispatch_failure count` |
| CDS publish | `cds.publish count, latency, failure count` |
| DB | per-table query/transaction metrics |
| Idempotency | `idempotency.cache_hit count`, `idempotency.payload_mismatch count` |

Stuck-event count, signature-failure rate, and `subscriber.dlq_count` are **page-worthy** alerts.

### 6.2.3 Tracing

- `correlation_id` ([§5.2](05-cross-cutting.md)) threads through orchestrations and downstream events. End-to-end traces should follow it.
- Bridge sets `correlation_id` from the inbound Beckn `transaction_id` for buyer-initiated flows.
- Admin actions set a fresh `correlation_id` per use-case invocation.

### 6.2.4 Health checks

- **Liveness** — process is responsive.
- **Readiness** — dependencies reachable (DB, message bus, IdP discovery doc, CDS, registry).
- **Bridge specifics**: signing key loaded, registry credentials valid, CDS reachable, last-successful-publish timestamp fresh enough.

### 6.2.5 Audit vs. observability — keep separate

- **Audit** ([§4.7](04-bounded-contexts/4.7-audit.md)) is a domain context with compliance retention. Don't replace it with a log retention policy.
- **Logs** are operational and may be aggressively pruned.
- **Don't mine logs for compliance answers** — query Audit instead.

---

## 6.3 Deployment topology

The charter defers topology ([§2.9](02-principles.md)). What's **non-negotiable regardless**:

1. **Layer boundaries are enforced at the source level**, not by network distance.
2. **Context boundaries are enforced by explicit contracts** (ports + events), not by deployment.
3. **The Bridge is isolatable** — it must be possible to deploy or replace it independently, even if today it lives in the same process as everything else.

### 6.3.1 v1 default — modular monolith

Reasonable v1 deployment:

- **One application** containing all seven contexts + Bridge + Admin UI + first-party APIs.
- **One database** with separate schemas per context (or, equivalently, table-prefix discipline).
- **In-process** event dispatcher (the outbox is a regular table; the dispatcher polls and calls subscribers directly).
- **Background workers** (cron, queue consumers) live in the same deploy unit or as sidecars.

Why this works:
- Lower operational overhead for v1.
- All the inter-context discipline ([§5.3 ports](05-cross-cutting.md)) is in place, so extraction later is mechanical.
- The Bridge being source-isolatable means swapping it (or running it standalone) is still tractable.

### 6.3.2 Source-level enforcement

Even in a monolith, **enforce boundaries at the build level**:

- Each context is its own module / package.
- Cross-context dependencies are allowed only on the **callee's published port interfaces**, never on internal types.
- The Domain Layer of each context never imports anything outside that context (per the Dependency Rule, [§2.3](02-principles.md)).
- The Bridge imports only Application-Layer port interfaces — never domain types, never persistence, never UI.
- A lint rule / build check verifying these import constraints is recommended.

### 6.3.3 Service extraction path

If the team eventually needs to extract a service, the cheapest candidates:

| Candidate | Why |
|---|---|
| **Beckn Bridge** | Already isolated by design; protocol traffic justifies independent scaling. |
| **Audit** | Append-only, write-heavy; can become a sink behind its own queue with no domain ties. |
| **Catalog read-side** | Browse traffic dwarfs writes once live; a read-side projection service is a natural split. |

**Mechanics for extraction**:
1. Move the context to its own DB / schema.
2. Replace in-process port resolution with RPC / HTTP / message-bus adapters.
3. Update deployment manifests; no calling code changes (per [§5.3.7](05-cross-cutting.md)).
4. Update observability (cross-process tracing) — `correlation_id` propagation across the boundary is a build-time change in transports.

### 6.3.4 Database posture

| Choice | Posture |
|---|---|
| **Database engine** | Implementing team's choice — Postgres is a reasonable default. |
| **Cross-context joins** | Forbidden ([§2.4](02-principles.md)) — even if the engine allows them. |
| **Foreign keys across contexts** | Forbidden — identifiers cross by value, not by FK. |
| **Outbox / inbox / idempotency** | All per-context tables. |

---

## 6.4 External dependencies

What the system relies on outside its own boundary:

| Dependency | Purpose | Replaceable? |
|---|---|---|
| **IdP** (OIDC-compatible) | Authentication, user identity claims | Yes — adapter pattern; provider-agnostic ([ADR-0009](../decisions/0009-identity-and-external-idp.md)) |
| **Beckn registry** | Network identity verification, signing-key lookup | Network-mandated — per Beckn protocol |
| **CDS endpoint(s)** | Catalog publishing (per [ADR-0017](../decisions/0017-order-and-fulfillment.md)) | Network-mandated |
| **Email delivery** | Invitations, notifications | Yes — provider-agnostic |
| **Object storage** | Media (per [§4.3](04-bounded-contexts/4.3-catalog.md)) | Yes — provider-agnostic |
| **Database** | Primary persistence | Yes — engine-agnostic per the Dependency Rule |
| **Message bus** (if extracted) | Outbox dispatcher transport | Yes — only relevant in distributed deployment |
| **Observability backend** | Logs/metrics/traces aggregation | Yes — vendor-agnostic |
| **Payment gateway** | Not in v1 — payment is external ([§4.6](04-bounded-contexts/4.6-order.md)) | Future |
| **Logistics provider** | Not in v1 — self-fulfilled ([§4.6](04-bounded-contexts/4.6-order.md)) | Future |

### 6.4.1 IdP integration

- OIDC discovery doc URL → IdP-agnostic.
- Claims required: `sub`, `email`, `email_verified`, `name`, optional `picture`, optional `locale`.
- Configurable provider (Clerk, Auth0, Supabase Auth, Cognito, Keycloak, …) per [ADR-0009](../decisions/0009-identity-and-external-idp.md).

### 6.4.2 Beckn network integration

From [ADR-0001](../decisions/0001-bpp-network-identity.md):

- `bpp-id` — platform identifier on the Beckn network.
- `bpp-uri` — single platform-wide callback endpoint.
- **Signing key material** — single platform key, stored securely; rotation is an operational procedure.
- **Registry credentials** — for verifying inbound participants and looking up keys.

From [ADR-0017](../decisions/0017-order-and-fulfillment.md):

- **CDS endpoint(s)** — where the Bridge publishes catalog updates.
- **CDS credentials** — auth for publish calls.

All Beckn network configuration is **operational** — the domain has no knowledge of it.

---

## 6.5 Configuration and secrets

### 6.5.1 Configuration layering

- **Compile-time**: feature flags, role/capability matrix structure (the catalog is code-defined; per-role grants are runtime — see [§5.1.3](05-cross-cutting.md)).
- **Deploy-time**: environment-specific config — DB connection strings, IdP URLs, CDS endpoints, message-bus endpoints, etc.
- **Runtime**: role → capability matrix grants (System Admin via [§5.1](05-cross-cutting.md)); voucher / catalog / Org content (Org Owners and Store Admins); operational toggles (rate limits, retention windows).

### 6.5.2 Secrets

- **Never in code, never in env-checked-in files.**
- **Secret-management service** (vault, AWS Secrets Manager, GCP Secret Manager, etc.) — implementing team's choice.
- **System Admin seed credentials** ([§5.1.1](05-cross-cutting.md)) — distributed via secure operational channel; never via in-band recovery.
- **Beckn signing key** — single platform key; rotation procedure is operational.
- **IdP client credentials** — confidential client; per environment.
- **CDS and registry credentials** — per environment.

### 6.5.3 Per-environment isolation

- Dev / staging / prod must use **distinct credentials** for IdP, CDS, registry.
- Dev / staging should never accept production Beckn traffic.
- Production must reject test-issued Beckn signatures.

---

## 6.6 Rate limiting and resource constraints

Rate limiting is **operational, not architectural** — but a few places need it specifically:

| Surface | Recommended limiting |
|---|---|
| Manual catalog republication (per [ADR-0019](../decisions/0019-manual-catalog-republication.md)) | Per-store rate limit (e.g., 1/min); per-Org rate limit on `Org.RequestRepublishAll` |
| Beckn `/select` callback rate | Per-BAP rate limit |
| Inbound Beckn signature verification | Bounded; failure rate is a metric |
| Admin login attempts | IdP-side concern, but Application Layer also rate-limits high-risk actions |
| Invitation email sends | Per-Org daily cap |
| Voucher attempts at quote time | Per-buyer rate limit (anti-abuse) |
| Audit search / list queries | Per-user query budget |

These are **defaults to consider**, not enforced architecture. Tune to traffic.

### 6.6.1 Resource constraints to plan for

- **Outbox table growth** — pruning needed (per [§5.2.6](05-cross-cutting.md)); default 90 days.
- **Inbox / `processed_events` growth** — per-subscription pruning.
- **Idempotency records** — 24h default ([§5.4.3](05-cross-cutting.md)).
- **Audit growth** — significantly larger than the outbox; sized for compliance retention ([§4.7](04-bounded-contexts/4.7-audit.md)).
- **Media storage** — Catalog images; lifecycle policies recommended (e.g., orphaned media after Product archive eventually removed by operational cleanup).

---

## 6.7 Operational runbooks (what the team will need)

A non-exhaustive list of runbooks the operating team should author. Each becomes valuable the first time it's needed.

### Identity & Access
- Provision / disable / reactivate a User (System Admin).
- Process a `ScrubUser` (right-to-erasure) request.
- Rotate IdP client credentials.
- Force sign-out a User (sign-out-everywhere).

### Tenancy
- Create an Organization (operational onboarding flow).
- Transfer Org ownership.
- Suspend / re-activate a Store from the platform side.
- Trigger `RepublishAll` for an Org (post-incident catalog refresh).

### Beckn / Bridge
- Rotate the platform signing key.
- Re-register with the Beckn registry.
- Replay catalog publishes after CDS outage.
- Investigate a signature-failure spike.
- Switch protocol-version mappers (during Beckn version migration).

### Events / consistency
- Investigate a stuck-events alert.
- Skip-with-acknowledgement procedure for an unfixable stuck event.
- Replay events for a subscriber from a known offset.
- Reset an inbox after a subscriber data migration.

### Audit
- Generate a compliance report (per Org, per Store, per User).
- Process a deletion sweep (expired audit records).
- Handle a subpoena / data-access request.

### Inventory / Catalog
- Bulk-import a catalog (System Admin / operational, not exposed in v1 UI).
- Bulk-adjust stock (after physical-count reconciliation).
- Handle a category-deprecation migration (System Admin).

### Database / Infrastructure
- Backup / restore.
- Schema migrations (zero-downtime).
- Failover.

---

> **Next**: [§7 Open issues and known limits](07-open-issues.md) — what's deferred from v1 and why.
