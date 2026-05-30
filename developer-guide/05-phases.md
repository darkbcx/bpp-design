# 5. Implementation phases

A **dependency-ordered build plan** for v1. Each phase produces something demo-able and is the prerequisite for the next. Phases are stack-agnostic — they describe what to build, not which library implements it.

This guide does not estimate calendar time; phase duration depends on team size and stack familiarity. What it locks in is **order** and **definition of done** per phase.

| Section | Topic |
|---|---|
| 5.1 | Reading this roadmap |
| 5.2 | Phase overview table |
| 5.3 | Phase 0 — Bootstrapping |
| 5.4 | Phase 1 — Foundations (cross-cutting + Identity) |
| 5.5 | Phase 2 — Tenancy |
| 5.6 | Phase 3 — Catalog |
| 5.7 | Phase 4 — Inventory + Promotion (parallel) |
| 5.8 | Phase 5 — Order & Fulfillment |
| 5.9 | Phase 6 — Beckn Bridge |
| 5.10 | Phase 7 — Audit maturity |
| 5.11 | Phase 8 — Hardening |
| 5.12 | Admin UI track (cross-phase) |
| 5.13 | Phase-completion checklist template |
| 5.14 | Common gotchas across phases |

---

## 5.1 Reading this roadmap

Each phase section gives:

- **Goal** — what's demo-able when the phase is done.
- **Depends on** — phases that must be complete first.
- **Handoff sections referenced** — where to read the architectural rules.
- **Deliverables** — concrete artifacts (entities, use cases, events, surfaces).
- **Definition of done** — testable criteria for "phase complete."
- **Watch for** — phase-specific gotchas.

Some work runs in parallel — the **Admin UI track** ([§5.12](#512-admin-ui-track-cross-phase)) streams alongside backend phases. **Audit subscription infrastructure** is set up in Phase 1; Audit's own context-specific features mature in Phase 7.

---

## 5.2 Phase overview table

| Phase | Goal | Depends on | Demo state |
|---|---|---|---|
| 0 | Bootstrapping | — | Repo skeleton, CI green, dev environment runs locally |
| 1 | Foundations + Identity | 0 | A User can sign in and see a "no Orgs yet" screen |
| 2 | Tenancy | 1 | A User can create an Org, invite Members, create Stores (Draft) |
| 3 | Catalog | 2 | A Store Admin can author and publish Products + Catalogs |
| 4 | Inventory + Promotion | 3 | Stock is tracked; Vouchers can be created and validated |
| 5 | Order & Fulfillment | 4 | Orders can be placed and progressed through state machine (via internal API) |
| 6 | Beckn Bridge | 5 | A simulated BAP can complete a Beckn v2 transaction end-to-end |
| 7 | Audit maturity | 1 (subscription) + others (events) | Audit records readable per access matrix; PII scrub works |
| 8 | Hardening | all prior | Production-ready: observability live, runbooks written, security reviewed |

Audit subscription writes start in Phase 1 (so events from Phase 2 onwards are captured); the **read side and access controls** mature in Phase 7.

---

## 5.3 Phase 0 — Bootstrapping

**Goal**: the BPP package's skeleton exists in the (host) monorepo; CI runs typecheck + lint + tests + build on every PR; a developer can clone, install, and run a placeholder service.

**Depends on**: nothing.

**Handoff sections referenced**: [§6.1 Testing](../handoff/06-operational.md), [§6.3 Deployment topology](../handoff/06-operational.md), [§6.5 Configuration and secrets](../handoff/06-operational.md).

### Deliverables

- **BPP package layout** within the monorepo (see [`02-repo-layout.md`](02-repo-layout.md)).
- **Lint rules** enforcing bounded-context boundaries (`import/no-restricted-paths` or equivalent — see [§5.3.7 of handoff](../handoff/05-cross-cutting.md)).
- **Shared base packages**:
  - **Contracts package** — Zod schemas for cross-package shapes.
  - **Kernel / application package** — the use-case envelope abstraction (validate → authorize → idempotency → transaction → emit → return), plus base error types.
  - **Value objects package** — `Money`, `LocalizedText`, `BCP47Tag`, `ISO4217Code`, `Email`, `Slug`, opaque IDs.
- **Database scaffold** — connection pool, migration runner, baseline empty migration.
- **Test infrastructure** — unit-test runner; integration-test runner with real DB (via Testcontainers or monorepo equivalent); end-to-end harness skeleton.
- **CI pipeline** — typecheck, lint, test (unit + integration), build, dependency audit.
- **Configuration layering** — compile-time, deploy-time, runtime ([§6.5.1 of handoff](../handoff/06-operational.md)). Secrets management hookup.
- **Pick remaining D-decisions** that block downstream phases: at minimum **D0** (DI container) and **D1** (query layer) — both needed before any context can ship.

### Definition of done

- A no-op HTTP route returns 200 on a fresh checkout + install + start.
- `pnpm test` (or equivalent) passes with zero tests written.
- Boundary lint rule fails CI on a deliberate cross-context internal-type import.
- A migration can be created, applied, and rolled back locally.
- Environment-distinct credentials are loaded from the secrets layer (not committed).

### Watch for

- **Boundary lint must be in place before Phase 1.** Adding it later is harder; introduce it when there's nothing to break.
- **Value objects are domain-language, not transport.** `Money.amount` is a bigint; `LocalizedText.entries` is keyed by BCP 47 tag. Don't model these as flat strings.
- **Idempotency key, correlation_id, active_org_id, active_store_id** should be wired into the request scope in Phase 0 or 1 — adding ambient context after routes exist is painful.

---

## 5.4 Phase 1 — Foundations + Identity

**Goal**: a real User can sign in via the IdP, gets a Session, and sees a "no Orgs yet" landing page. Cross-cutting infrastructure (outbox / inbox / idempotency / authorization / event dispatcher) is live and exercised by Identity.

**Depends on**: Phase 0.

**Handoff sections referenced**: [§4.1 Identity](../handoff/04-bounded-contexts/4.1-identity.md), [§5.1 Authorization](../handoff/05-cross-cutting.md), [§5.2 Domain events](../handoff/05-cross-cutting.md), [§5.3 Cross-context consistency](../handoff/05-cross-cutting.md), [§5.4 Idempotency](../handoff/05-cross-cutting.md), [§5.6 PII](../handoff/05-cross-cutting.md).

### Deliverables

**Cross-cutting**:
- **Outbox** table + dispatcher (in-process). Subscriber registration mechanism. `LISTEN`/`NOTIFY` wake + interval polling.
- **Inbox** dedup pattern (`processed_events` per subscription).
- **Idempotency-key middleware** + `idempotency_records` table. Conflict detection (same key + different payload → typed error).
- **`AuthorizationPort`** + capability matrix scaffold ([§5.1.4 of handoff](../handoff/05-cross-cutting.md)). Hardcoded role grants for v1 (matrix-editing UI lands in Phase 8). `requireCapability(name, scope)` at the top of every mutating use case.
- **Redacting logger** ([§5.6.6 of handoff](../handoff/05-cross-cutting.md)) consuming the PII catalog from `design/pii.md`.
- **Per-request DI scope** carrying `correlation_id`, `active_org_id`, `active_store_id`, actor reference.
- **Audit subscriber stub** subscribed broadly — at this phase, it writes `AuditRecord`s but no read UI exists yet (full Audit context lands in Phase 7).

**Identity & Access context**:
- `User` entity (IdP-canonical + platform-owned fields per [§4.1 of handoff](../handoff/04-bounded-contexts/4.1-identity.md)).
- `Session` entity + opaque-token storage + signed cookie.
- IdP integration (OIDC client, sign-in flow, session creation).
- **JIT provisioning** — first sign-in creates the `User` record.
- Use cases: `SignIn`, `SignOut`, `SignOutEverywhere`, `GetCurrentUser`.
- Events: `identity.user_provisioned`, `identity.session_created`, `identity.session_terminated`, `identity.authorization_denied`.

**Admin UI (parallel)**: sign-in flow, "no Orgs yet" landing page, sign-out.

### Definition of done

- Real sign-in against a configured IdP sandbox; cookie set; protected route returns 200; sign-out invalidates the session.
- A use case decorated with `requireCapability` rejects an unauthorized actor with a typed error AND emits `identity.authorization_denied`.
- Pre-attached idempotency key on a mutating route causes a duplicate to return the cached result without state-mutating side effects.
- Outbox row is written in the same transaction as `identity.user_provisioned`; dispatcher delivers it to a stub subscriber; subscriber's inbox prevents double-processing on redelivery.
- Audit subscriber writes one `AuditRecord` per event with full envelope.
- Redacting logger masks email in logs by default.

### Watch for

- **Don't let "no Org" be a magic ambient state.** Treat `active_org_id = null` as a first-class condition (System Admin + Platform-scoped users live here permanently; Tenant-scoped users only transiently before joining an Org).
- **`email_verified_at` gating.** Acceptance of invitations, store creation, etc., requires `email_verified_at` set ([§4.1 of handoff](../handoff/04-bounded-contexts/4.1-identity.md)). If the IdP says verified, trust it.
- **Outbox claims must use `FOR UPDATE SKIP LOCKED`** (or equivalent). Without it, multiple dispatcher workers will fight over the same rows.
- **Idempotency records belong to the same DB transaction as the state mutation**, not a separate write. A pre-write record causes ghost records on crash.

---

## 5.5 Phase 2 — Tenancy

**Goal**: a User can create an Organization, invite Members, switch between Orgs they belong to, create Stores within the active Org, and assign Store Admins. Stores live in their lifecycle states (`Draft` / `Active` / `Suspended` / `Paused`) per [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md).

**Depends on**: Phase 1.

**Handoff sections referenced**: [§4.2 Tenancy](../handoff/04-bounded-contexts/4.2-tenancy.md), [§5.1.5 Active Org / Store](../handoff/05-cross-cutting.md), [§6 Invitation & Role of CLAUDE.md](../CLAUDE.md).

### Deliverables

- **Entities**: `Organization`, `OrganizationMember`, `OrganizationInvitation`, `Store`, `StoreAdminAssignment`.
- **Use cases**: `CreateOrg`, `InviteOrgMember`, `AcceptInvitation`, `DeclineInvitation`, `RevokeInvitation`, `RemoveOrgMember`, `TransferOrgOwnership`, `CreateStore`, `SubmitStoreForActivation`, `PauseStore`, `ResumeStore`, `AssignStoreAdmin`, `UnassignStoreAdmin`.
- **Platform-scoped use cases**: `ActivateStore`, `SuspendStore`, `ReactivateStore`.
- **Active Org / Active Store full integration**: session attribute, URL-encoded path `/orgs/<org-slug>/stores/<store-slug>/...`, mismatch detection.
- **Capability matrix populated** for `org.*` and `store.*` capabilities (per [§5.1.2 of handoff](../handoff/05-cross-cutting.md)).
- **`StoreStatusChanged` event** + republication request events (`Store.RequestRepublish`, `Org.RequestRepublishAll`, per [ADR-0019](../decisions/0019-manual-catalog-republication.md)). Bridge subscribes in Phase 6 — for now, just emit.
- **Email adapter** (port + concrete adapter) for invitation emails.

**Admin UI (parallel)**: Org switcher, Org settings, Member management, Invitation flow, Store creation form, Store list, lifecycle controls.

### Definition of done

- A new User → creates an Org → invites another User → recipient signs up via IdP → accepts → both see the Org in their switcher → both can see Stores within it.
- A Store goes Draft → submit → platform-activate → Active. Owner can Pause; only platform can Suspend.
- Invitation email is sent; Revoke + Expire transitions work; declined invitations don't create Memberships.
- Ownership transfer requires explicit acceptance by the new Owner (no unilateral pushes).
- URL path mismatch (`active_org` in session ≠ `<org-slug>` in URL) returns 403.
- All Tenancy mutations emit appropriate events; events flow through outbox → dispatcher → Audit subscriber → AuditRecord visible.

### Watch for

- **Store ownership lives at the Org level** ([ADR-0018](../decisions/0018-organization-tenancy.md)). There's no per-store Owner. Avoid copying the old per-store-Owner shape from earlier reference projects.
- **Invitation acceptance reconciliation** — match invitation email to IdP-canonical email exactly (case-insensitive). No claim flow with alternate email; mismatch = typed error.
- **`StoreStatusChanged` is the single trigger** for Bridge republication. Don't add other republication signals; the Bridge subscribes to this event (and the explicit `Store.RequestRepublish`).
- **Acceptance is idempotent.** Already-accepted invitations return the existing `OrganizationMember`; double-accept on a Pending invitation creates exactly one Member.

---

## 5.6 Phase 3 — Catalog

**Goal**: a Store Admin can author Products (Matrix or Flat mode), assign them to PlatformCategories, create additional Catalogs alongside the mandatory Default, manage media, and set pricing.

**Depends on**: Phase 2.

**Handoff sections referenced**: [§4.3 Catalog](../handoff/04-bounded-contexts/4.3-catalog.md), [`design/catalog.md`](../design/catalog.md), [ADR-0005](../decisions/0005-catalog-and-product-modeling.md), [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md), [ADR-0008](../decisions/0008-localization-and-localizedtext.md), [ADR-0020](../decisions/0020-default-catalog-mandatory.md).

### Deliverables

- **Entities**: `PlatformCategory` (hierarchical, System-Admin-managed), `Product` (Matrix or Flat), `ProductVariant`, `ProductAttribute`, `Catalog` (Default auto-created with Store; additional opt-in), `CatalogMembership` (for additional Catalogs; Default is implicit), `ProductCategoryAssignment`, `Media`.
- **Use cases**: full Product / Variant / Catalog / Category authoring + lifecycle. See [§4.3 of handoff](../handoff/04-bounded-contexts/4.3-catalog.md).
- **Pricing computation** — derived `tax_amount`, `published_price` cached on Product; recomputed on `base_price` or `tax_rate` change.
- **Object storage adapter** (D4) for Media — signed-URL upload pattern.
- **PlatformCategory taxonomy** authored as `LocalizedText`; missing translations fall back to default locale (`id`).
- **Auto-creation hooks**: creating a Store auto-creates its Default Catalog. Creating a Product auto-includes it in the Default.
- **Events**: `catalog.product_created`, `_updated`, `_published`, `_archived`, `_restored`, `_media_updated`, plus all the Variant / Attribute / Category / Catalog events from [`design/events.md`](../design/events.md).

**Admin UI (parallel)**: Product authoring (Matrix vs Flat picker), Variant management, media upload, Category picker, Catalog management.

### Definition of done

- A Store Admin creates a Matrix-mode Product with a Size/Color attribute set; variants are generated; per-variant media works.
- A different Product is created in Flat mode with its own SKU.
- A Product moves Draft → Active when all invariants are met (≥1 PlatformCategory, base_price + tax_rate set).
- `published_price` is correctly derived and recomputed on price/rate change.
- Default Catalog membership is automatic and immovable; a Product can be added to additional Catalogs and removed from them.
- All Product fields supporting `LocalizedText` accept multi-locale entries and fall back to `id` correctly.
- A second store cannot see the first store's Products.

### Watch for

- **`Product.mode` is immutable.** Once a Product is in Matrix mode, it stays Matrix; you can't add per-variant SKUs to it later. Surface this in the authoring UI.
- **`Product.name` uniqueness check is against the default-locale value** ([§5.7.5 of handoff](../handoff/05-cross-cutting.md)). Don't try to enforce cross-locale uniqueness.
- **Default Catalog inclusion is implicit**, not a `CatalogMembership` row ([ADR-0020](../decisions/0020-default-catalog-mandatory.md)). Don't try to "exclude from Default" — that's `Archive` the Product.
- **Variants inherit parent's `tax_rate`** ([ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)). Only `price_override` is per-variant. No per-variant tax.
- **`Store.currency` is immutable** once any Active Product exists. Surface this in Store settings (locked after first Active Product).

---

## 5.7 Phase 4 — Inventory + Promotion (parallel)

**Goal**: Stock levels are tracked per inventory item (Flat Product or Matrix Variant); Reservations can be created, converted, and released; Vouchers can be created, validated, and recorded as used.

**Depends on**: Phase 3.

**Handoff sections referenced**: [§4.4 Inventory](../handoff/04-bounded-contexts/4.4-inventory.md), [§4.5 Promotion](../handoff/04-bounded-contexts/4.5-promotion.md), [ADR-0006](../decisions/0006-inventory-model.md), [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md).

These two contexts are independent siblings — both depend on Catalog (Phase 3) and neither depends on the other. They can be built in parallel by two engineers / pairs.

### Deliverables — Inventory

- **Entities**: `StockLevel` (one per inventory item, auto-created when Catalog item is created), `Reservation`, stock-movement events.
- **Use cases**: `AdjustStock`, `SetPurchasable`, `Reserve`, `ConvertReservation`, `ReleaseReservation`, `GetAvailability`, `BulkGetAvailability`.
- **Subscription to Catalog lifecycle events** — `OnProductCreated`, `_Archived`, `_Restored`, `_VariantAdded`, `_VariantRemoved` — auto-manages `StockLevel` state.
- **Effective availability formula** computed at read time (per [§4.4 of handoff](../handoff/04-bounded-contexts/4.4-inventory.md)).
- **Events**: `inventory.stock_level_created`, `_inactivated`, `inventory.stock_received`, `_sold`, `_returned`, `_corrected`, `_reserved`, `_released`, `inventory.purchasable_toggled`.

### Deliverables — Promotion

- **Entities**: `Voucher` (store-scoped, code-based), `VoucherUsage` (redemption log).
- **Use cases**: `CreateVoucher`, `UpdateVoucher`, `DisableVoucher`, `EnableVoucher`, `ListVouchers`, `GetVoucherUsageHistory`. Cross-context: `ValidateVoucher` (used by Order in Phase 5), `RecordVoucherUsage`, `RevertVoucherUsage`.
- **Discount × tax interaction** — voucher discount applies to pre-tax `base_price`; tax recomputes on the discounted base.
- **One-voucher-per-order** rule enforced at validation time.
- **Events**: `promotion.voucher_created`, `_updated`, `_disabled`, `_enabled`, `_redeemed`, `_usage_reverted`.

**Admin UI (parallel)**: Inventory dashboard (stock levels, manual adjustment), Voucher CRUD, voucher usage report.

### Definition of done — Inventory

- Creating a Product auto-creates a `StockLevel = 0, purchasable = true`.
- Archiving a Product cascades `StockLevel.status = Inactive`; new reservations rejected.
- A successful `Reserve(item, qty)` decrements available quantity but not `stock_count`; on `ConvertReservation`, `stock_count` decrements atomically; `Released` reservations don't affect `stock_count`.
- An expired Reservation eventually releases (background sweep).
- Concurrent reserves on a near-empty StockLevel: at most `stock_count` succeed.

### Definition of done — Promotion

- `Voucher` with case-insensitive unique code per store.
- `ValidateVoucher` returns typed discount or typed error covering: expired, exhausted, store mismatch, status disabled, cart-value too low, per-user limit exceeded.
- `RecordVoucherUsage` is atomic with the Voucher's `current_total_uses` increment.
- `RevertVoucherUsage` decrements the counter and flags the usage row.
- Voucher discount applied to `base_price` correctly produces the expected `published_price` after tax recompute.

### Watch for

- **Reservations are owned by Order's transaction in Phase 5**, but the Reservation primitive itself is built in Phase 4. Don't bake Order-specific logic into Inventory.
- **`stock_count` only changes via Reservation conversion, manual `Corrected`, or `Received`** — never directly. Reservation-Active→Released does NOT decrement.
- **Voucher discount applies to `base_price`, not `published_price`** — GST/VAT-compliant. Tax recomputes on the discounted base.
- **`VoucherUsage` is the source of truth** for limits; `current_total_uses` on Voucher is a cache. Restore it from `VoucherUsage` count if drift occurs.

---

## 5.8 Phase 5 — Order & Fulfillment

**Goal**: an Order can move through the full lifecycle (`Created` → `Initiated` → `Confirmed` → `Fulfilled`) via internal API. Quote snapshots are captured at creation; Reservations are held and converted at the right boundaries; Vouchers are applied and reverted on cancel-from-Confirmed; fulfillment status transitions work.

**Depends on**: Phase 4 (both Inventory and Promotion).

**Handoff sections referenced**: [§4.6 Order & Fulfillment](../handoff/04-bounded-contexts/4.6-order.md), [`design/order.md`](../design/order.md), [ADR-0017](../decisions/0017-order-and-fulfillment.md), [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md), [§5.3 Cross-context consistency](../handoff/05-cross-cutting.md).

### Deliverables

- **Entities**: `Order` (state machine + payment_status + fulfillment_status), embedded `Quote`, embedded `LineItem`s (with captured snapshots).
- **Use cases** (the orchestrations, all using Application-Layer ports with hand-coded compensation):
  - `Order.CreateQuote(store, items, voucher_code?, beckn_buyer_ref)` — calls Catalog (snapshot capture), Promotion (validate), Inventory (read availability only — no reserve yet).
  - `Order.Initiate(order, contact_snapshot)` — calls Inventory (Reserve), Promotion (re-validate); compensates by releasing on failure.
  - `Order.Confirm(order)` — calls Inventory (ConvertReservation), Promotion (RecordVoucherUsage).
  - `Order.Cancel(order, reason)` — releases reservations OR reverts voucher + marks payment Refunded based on current state.
  - `Order.GetStatus(order)`.
  - `Order.MarkPreparing`, `_Shipped`, `_Fulfilled` — fulfillment progression.
- **Quote TTL** (default 15 min; operational config) — background sweep moves expired Orders to `Expired`, releases reservations if held.
- **Buyer model**: Beckn-typed only ([ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)). `bap_id` + `transaction_id` at Created; `contact_snapshot` populated at Initiated.
- **Captured snapshots** on `LineItem`: name, description (LocalizedText), base_price, tax_rate, derived tax_amount and published_price.
- **Payment status tracking** — driven by external signals (Beckn-side or, later, gateway webhooks).
- **Events**: `order.quote_created`, `_initiated`, `_confirmed`, `_cancelled`, `_expired`, `_marked_preparing`, `_marked_shipped`, `_fulfilled`, `_payment_status_changed`. Plus `promotion.voucher_usage_reverted` (emitted by Promotion when Order cancels post-Confirm).

**Admin UI (parallel)**: Order list (filter by store, status), Order detail view, Fulfillment action buttons (Mark Preparing / Shipped / Fulfilled), Cancel action.

### Definition of done

- Full flow via internal API (no Beckn yet): create quote → initiate → confirm → mark preparing → shipped → fulfilled.
- Quote TTL expires correctly; reservations release.
- Cancel from `Initiated` releases reservations; cancel from `Confirmed` reverts voucher and marks payment Refunded.
- Captured snapshots don't refresh when Catalog prices change after Quote creation.
- Compensation on partial failure works: a failed `ValidateVoucher` during Initiate releases all the Reservations created in the same call.
- Audit records show actor + impersonator attribution correctly on every Order mutation.

### Watch for

- **No first-party Order entry in v1** ([ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)). The Order use cases are designed to be called only by the Bridge (Phase 6). Don't expose a first-party API surface for buyer-side Order placement.
- **Compensation is hand-coded, not Saga-managed** ([§5.3.3 of handoff](../handoff/05-cross-cutting.md)). For each multi-context call, write the `try ... on failure ...` explicitly.
- **Stock is NOT returned on cancel-from-Confirmed in v1**. Return / refund post-fulfillment is deferred ([§7.2.6 of handoff](../handoff/07-open-issues.md)).
- **Quote snapshots are immutable**. Don't add "refresh quote" logic — Catalog price changes mid-Quote produce a stale snapshot that's intentional.
- **Payment status is external** — BPP doesn't capture funds. `payment_status` updates come from outside (BAP or future gateway).

---

## 5.9 Phase 6 — Beckn Bridge

**Goal**: a simulated BAP can complete a full Beckn v2 transaction end-to-end. Catalog updates publish to CDS. Inbound `/select`, `/init`, `/confirm`, `/status`, `/cancel` work; outbound `/on_*` callbacks correlate correctly.

**Depends on**: Phase 5 (Order use cases callable via Bridge); Phase 3 (Catalog ready to project); Phase 2 (Store / StoreStatusChanged events).

**Handoff sections referenced**: [§3 Beckn integration](../handoff/03-beckn-integration.md), [§4.6 Order](../handoff/04-bounded-contexts/4.6-order.md), [ADR-0001](../decisions/0001-bpp-network-identity.md), [ADR-0017](../decisions/0017-order-and-fulfillment.md), [ADR-0019](../decisions/0019-manual-catalog-republication.md), [`design/order.md`](../design/order.md).

### Deliverables

- **Bridge adapter** in the Interface Layer — the sole place Beckn vocabulary is permitted ([§4 of CLAUDE.md](../CLAUDE.md)).
- **Network identity** ([ADR-0001](../decisions/0001-bpp-network-identity.md)): `bpp-id`, `bpp-uri`, platform signing key, registry credentials — all from configuration.
- **Inbound handlers**: `/select`, `/init`, `/confirm`, `/status`, `/cancel`. For each: signature verification, schema validation, transaction_id correlation, extract domain intent, invoke the appropriate Application-Layer use case.
- **Outbound dispatch**: `/on_select`, `/on_init`, `/on_confirm`, `/on_status`, `/on_cancel` callbacks. Build wire payloads from domain results; sign; dispatch to BAP callback URL.
- **Mapping registry** — explicit, documented, testable. Order ↔ Contract; LineItem ↔ Commitment; fulfillment_status ↔ Performance; Voucher ↔ Offer; etc.
- **CDS publish adapter** — subscribed to `tenancy.store_status_changed`, all `catalog.*` mutation events, manual `Store.RequestRepublish` / `Org.RequestRepublishAll` events. Publishes to the configured CDS endpoint. Idempotent re-projection.
- **Wire-state mapping**: `Active` → `Catalog.isActive: true`; `Paused` / `Suspended` → `Catalog.isActive: false`; `Draft` → not published.
- **Multi-catalog projection** — each Catalog (Default + additional) projects as a distinguishable grouping under the Provider.
- **Beckn-level idempotency** — protocol-level duplicate inbound messages detected by `(transaction_id, message_id)` and short-circuited before reaching the Application Layer.
- **Locale handling** — per [§3.4.6 of handoff](../handoff/03-beckn-integration.md), request locale resolved through `LocalizedText.get`; no automatic translation.
- **Error mapping** — domain errors → Beckn error codes via the mapping registry. Unmappable domain errors fall through to a generic protocol error AND log the mismatch.

### Definition of done

- A simulated BAP issuing `/select` against a store with Active products receives a valid `/on_select` callback within the expected window. Quote in the callback matches the domain Quote.
- `/init` populates `contact_snapshot` on the Order, holds Reservations, and replies with `/on_init` containing the populated Contract.
- `/confirm` converts Reservations, records voucher usage, replies with `/on_confirm`.
- `/cancel` releases or reverts based on state.
- Catalog publish to CDS happens automatically on every relevant event AND on manual `RequestRepublish`. Re-projection is idempotent.
- Wire-state mapping correct across all Store lifecycle transitions.
- Signature verification rejects invalid signatures; idempotency at the protocol boundary recognizes duplicates by `(transaction_id, message_id)`.
- The Application Layer remains Beckn-naive: no Beckn vocabulary in domain code; no Beckn payloads in non-Bridge tests.

### Watch for

- **The Bridge is the only place Beckn vocabulary may appear.** If you find yourself adding `descriptor` or `provider` to a domain entity, stop — that's protocol leakage.
- **Beckn is asynchronous, callback-driven.** Acknowledge inbound requests immediately; deliver substantive responses via callbacks. Correlation by `transaction_id` is the Bridge's job, not the Application Layer's.
- **Errors must be translated in both directions.** Domain errors become Beckn error codes; Beckn protocol errors are handled inside the Bridge and never surface as domain errors to inner layers.
- **Multi-catalog projection has Bridge-only mapping** ([§3.4.1 of handoff](../handoff/03-beckn-integration.md)). The domain has `Catalog` entities; the wire has whatever structure the mapping registry decides.
- **CDS is mandated**, BPP does NOT field `/discover` directly ([ADR-0017](../decisions/0017-order-and-fulfillment.md)). The discovery model is BAP → CDS → BPP transactional only.
- **Registry lifecycle is operational**, not in-band. The Bridge consumes registry credentials but doesn't register / unregister itself.

---

## 5.10 Phase 7 — Audit maturity

**Goal**: Audit's read side, access controls, retention sweeps, and PII scrub-in-place are fully implemented. Audit has been writing records since Phase 1; now it becomes consumable.

**Depends on**: Phase 1 (subscription); Phase 2+ (events being generated).

**Handoff sections referenced**: [§4.7 Audit](../handoff/04-bounded-contexts/4.7-audit.md), [§5.5 Soft-delete](../handoff/05-cross-cutting.md), [§5.6 PII](../handoff/05-cross-cutting.md), [ADR-0014](../decisions/0014-soft-delete-and-audit.md), [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md), [`design/audit.md`](../design/audit.md), [`design/pii.md`](../design/pii.md).

### Deliverables

- **DB role separation**: `audit_subscriber` (INSERT only), `audit_pii_scrubber` (UPDATE on `source_envelope` only), `audit_cleanup` (DELETE only, row-filtered), `audit_reader` (SELECT only).
- **Read use cases** with access matrix enforcement: `GetAuditRecord`, `ListRecordsForStore`, `ListRecordsForUser`, `SearchRecords` (platform-scope).
- **Retention sweep job** — scheduled cleanup per category (financial 7y; inventory 3y; business state 2y; identity 1y; catalog routine 1y; per `design/audit.md` / operational config).
- **PII catalog implementation** — code-level registry mirroring `design/pii.md`. Used by the redacting logger AND by the scrub workflow.
- **`ScrubUser(user_id, by_actor, reason)`** orchestration ([§5.6.2 of handoff](../handoff/05-cross-cutting.md)): walks all contexts via scrub ports; scrubs PII in place per the catalog; scrubs `source_envelope` fields via `audit_pii_scrubber`; emits `identity.user_pii_scrubbed`.
- **Auto-scrub on Disable** — configurable window (default off).

**Admin UI (parallel)**: Audit log viewer per Store (Org Owner / Store Admin), per User (the User's own activity), platform-wide search (System Admin / capable Platform-scoped).

### Definition of done

- A Store Admin can read AuditRecords for their store but not another store's.
- A User can read their own activity records, including impersonation events where they were the impersonated party.
- A System Admin can read platform-wide.
- Retention sweep deletes only records past their category's retention; nothing else can delete.
- `ScrubUser` on a test user: PII fields in User, Tenancy, Order, and Audit `source_envelope` are all replaced with deterministic placeholders; `User` record itself stays; foreign references stay valid.
- After scrub, the `audit_subscriber` role still cannot UPDATE; `audit_pii_scrubber` can UPDATE only the field-list it has permission for.

### Watch for

- **Audit is append-only at the DB role level**, not by convention ([§4.7 of handoff](../handoff/04-bounded-contexts/4.7-audit.md)). Use real DB roles; don't enforce in application code only.
- **PII scrubbing is the only permitted UPDATE on audit records.** Other "fix the audit log" requests get a hard no — that's the architectural commitment.
- **Cryptographic chaining is deferred to a future ADR** ([§4.7 deferred items](../handoff/04-bounded-contexts/4.7-audit.md)). Don't pre-build it.
- **Retention is per category, not per record.** Categorize each event type when the event lands in the registry, not at audit-read time.

---

## 5.11 Phase 8 — Hardening

**Goal**: production-ready posture: full observability stack live; performance baseline + load tests; security review; operational runbooks written; field-level encryption applied where compliance demands.

**Depends on**: all prior phases.

**Handoff sections referenced**: [§6 Operational stance](../handoff/06-operational.md), [§5.6.9 Encryption](../handoff/05-cross-cutting.md), [§7 Open issues](../handoff/07-open-issues.md).

### Deliverables

- **Observability live**: OpenTelemetry SDK wired; logs + metrics + traces flowing to the chosen backend (D5); standard metrics families per [§6.2.2 of handoff](../handoff/06-operational.md) emitting.
- **Page-worthy alerts**: stuck-events count, signature-failure rate, subscriber DLQ count.
- **Performance baseline + load tests** at projected v1 traffic. Document p50/p95/p99 for the key Beckn flows.
- **Security review**: OWASP Top 10 pass; PDP compliance audit; signing-key handling review; secret-management audit.
- **Field-level encryption** applied to specific fields per `design/pii.md` if compliance requires (uses Postgres `pgcrypto` or equivalent).
- **Runbooks written** per [§6.7 of handoff](../handoff/06-operational.md): Identity (scrub, force sign-out, IdP rotation), Tenancy (Org transfer, Store activation), Bridge (signing-key rotation, registry re-registration, catalog replay, signature-failure investigation, version-mapper switch), Events (stuck-events triage, skip-with-ack, replay), Audit (compliance report, deletion sweep, subpoena), Inventory / Catalog (bulk import, reconciliation, category-deprecation), DB / Infra (backup, restore, schema migrations, failover).
- **System Admin matrix-editing UI** — the in-band tool for managing role → capability grants ([§5.1.3 of handoff](../handoff/05-cross-cutting.md)).
- **Stuck-events admin tooling** — UI for review, retry, skip-with-acknowledgement.
- **Documentation pass** — keep `developer-guide/` aligned with the actual code; update README and runbooks as needed.

### Definition of done

- Alerts page someone on simulated stuck-event scenarios.
- Backup → restore drill executed successfully.
- A simulated signing-key rotation completes without disrupting in-flight Beckn traffic.
- Performance baseline documented; no regression alerts during 1-hour synthetic load.
- All runbooks rehearsed at least once by someone other than the author.
- A System Admin can edit the capability matrix in-band; changes audit cleanly.

### Watch for

- **Audit and observability are separate planes** ([§4.7 of handoff](../handoff/04-bounded-contexts/4.7-audit.md), [§6.2.5 of handoff](../handoff/06-operational.md)). Don't try to use observability logs as a compliance source.
- **Page-worthy alerts should be few and meaningful.** Tune them to suppress noise; on-call fatigue is a real risk.
- **The PII catalog drives both the redacting logger and the scrub workflow.** Adding a new PII field in any context requires updating the catalog AND testing both consumers.

---

## 5.12 Admin UI track (cross-phase)

The Admin UI is its own track — it can start as soon as Phase 1 provides auth + session. Each backend phase ships its admin surfaces alongside the context being built.

| Phase | Admin UI deliverable |
|---|---|
| 1 | Sign-in, "no Orgs yet" landing, sign-out |
| 2 | Org switcher, Org settings, Member management, Invitations, Store list / create / lifecycle |
| 3 | Catalog UI: Product authoring (Matrix vs Flat), Variant management, Categories, additional Catalogs, media upload |
| 4 | Inventory dashboard (stock levels, manual adjustment); Voucher CRUD + usage report |
| 5 | Order list, Order detail, Fulfillment actions, Cancel |
| 6 | No admin surface (Bridge is wire-only) — though a "Bridge health" panel could land in Phase 8 |
| 7 | Audit log viewer (per Store, per User, platform-wide search) |
| 8 | System Admin matrix-editing; stuck-events tooling; operational dashboards |

The Admin UI can be staffed by separate engineers from backend; the contracts package keeps both sides honest.

### URL hierarchy ([§5.1.5 of handoff](../handoff/05-cross-cutting.md))

```
/                                                root (after sign-in)
/orgs                                            org switcher
/orgs/<org-slug>/                                org dashboard
/orgs/<org-slug>/members
/orgs/<org-slug>/invitations
/orgs/<org-slug>/stores
/orgs/<org-slug>/stores/<store-slug>/            store dashboard
/orgs/<org-slug>/stores/<store-slug>/catalog/products
/orgs/<org-slug>/stores/<store-slug>/catalog/catalogs
/orgs/<org-slug>/stores/<store-slug>/inventory
/orgs/<org-slug>/stores/<store-slug>/vouchers
/orgs/<org-slug>/stores/<store-slug>/orders
/orgs/<org-slug>/stores/<store-slug>/audit
/platform/                                       Platform-scoped + System Admin entry
/platform/categories                             System Admin: PlatformCategory taxonomy
/platform/matrix                                 System Admin: role → capability matrix
/platform/audit                                  cross-tenant audit search
```

URL slugs MUST match session values for `active_org_id` / `active_store_id` on every request; mismatch is an authorization failure.

---

## 5.13 Phase-completion checklist template

For each phase, the team should be able to tick:

- [ ] All deliverables landed.
- [ ] All "Definition of done" items pass automated tests.
- [ ] No cross-context internal-type imports (verified by lint).
- [ ] All new events present in the event registry ([`design/events.md`](../design/events.md)).
- [ ] All new PII fields present in the PII catalog ([`design/pii.md`](../design/pii.md)).
- [ ] All new capabilities present in the capability catalog (the code-side mirror of [§5.1.2 of handoff](../handoff/05-cross-cutting.md)).
- [ ] All new use cases start with `requireCapability(...)`.
- [ ] All new mutating use cases support `Idempotency-Key`.
- [ ] All new state-change use cases emit events to the outbox in the same transaction.
- [ ] Cross-context calls go through Application-Layer ports (no internal-type imports).
- [ ] Audit subscriber writes records for all new events (verify with one test per new event).
- [ ] Beckn vocabulary appears nowhere outside the Bridge package (verified by lint OR manual grep).

---

## 5.14 Common gotchas across phases

A non-exhaustive list of mistakes that bite multiple phases. Watch for these every phase.

| Mistake | Where to look for it |
|---|---|
| Beckn vocabulary leaks into Domain or Application | Phase 3+ (Catalog), Phase 5+ (Order); always check Phase 6 doesn't leak into prior |
| Idempotency record written outside the state-mutation transaction | Phase 1 and every Phase that adds mutating use cases |
| `requireCapability` missing at the top of a use case | Phase 2+ — every new use case |
| Outbox row written outside the state-mutation transaction | Phase 2+ — every event-emitting use case |
| Beckn `transaction_id` correlation logic leaking out of Bridge | Phase 6 |
| Direct cross-context internal-type imports | Every phase — relies on lint rule from Phase 0 |
| Soft-delete becoming hard-delete by accident | Phase 2+ — every business entity |
| PII appearing in logs | Phase 1+ — every new logging call site |
| `LocalizedText` without an `id` entry | Phase 3+ — every translatable field |
| Stock decrement without going through Reservation conversion | Phase 4+ |
| Quote snapshot refreshed mid-flight | Phase 5 |
| Cross-context join in SQL | Every phase that adds DB queries |
| Beckn-version assumption baked into mapping | Phase 6 — every new mapping registry entry |

---

> **Next**: [`02-repo-layout.md`](02-repo-layout.md) — concrete package structure for the BPP within the host monorepo. After that, [`06-context-playbooks/`](06-context-playbooks/) for per-context implementation guides.
