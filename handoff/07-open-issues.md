# 7. Open issues and known limits

This section catalogs everything the design **deliberately defers** from v1. Each item has a sentence about *why* — so the implementing team can recognize when a request lands on a deferred feature and respond with intent rather than scope creep.

> If a v1 feature request lands on a deferred item, the correct response is **scope conversation**, not silent expansion. The deferral exists because the design carved a small, defensible v1 surface; adding any of these back during v1 changes the shape of the system.

| # | Topic |
|---|---|
| 7.1 | v1 scope baseline |
| 7.2 | Deferred per bounded context |
| 7.3 | Deferred cross-cutting concerns |
| 7.4 | Deferred operational capabilities |
| 7.5 | Known limits and assumptions |
| 7.6 | Migration paths for deferred features |

---

## 7.1 v1 scope baseline

What v1 **does**:

- Multi-tenant aggregator BPP for small retailers in Indonesia (PDP-compliant).
- Beckn v2 / ION wire vocabulary at the protocol boundary; CDS-mediated discovery.
- One Organization → many Stores; Stores publish as Beckn `Provider` nodes.
- Catalog with four product modes — `Standalone`, `Variant`, `Configurable` (modifier + configurator items), `Composite` (bundles); multi-catalog projection (Default mandatory + additional opt-in).
- Inventory with single-location stock, reservations, no backorders; two anchor types — `StockLevel` for stock-tracked items, `OwnerPurchasability` (purchasable flag only) for Composite + Configurable Products; per-mode effective-availability computation; reservation fan-out for Composite / Configurable confirms.
- Vouchers — store-scoped, code-based, one per order.
- Order flow Created → Initiated → Confirmed → Fulfilled, with Cancel and Expire terminals; pre-fulfillment cancel only.
- Self-fulfillment by stores (no platform logistics).
- External payment (no gateway integration in BPP).
- Admin UI for Org Owner / Store Admin / System Admin / Platform-scoped roles.
- Audit, PII scrubbing, idempotency, localization, eventual consistency across contexts.

What v1 **doesn't** — captured below.

---

## 7.2 Deferred per bounded context

### 7.2.1 Identity & Access ([§4.1](04-bounded-contexts/4.1-identity.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Multi-IdP linking (one User → multiple `external_subject_id`s, e.g., Google + Apple) | Adds an `ExternalIdentity` entity; v1 needs only single identity. | When a strategic IdP migration or co-login is required. |
| Service accounts / API tokens | First-party API access is admin-driven for v1; service-to-service is out of scope. | When third parties or partners integrate. |
| Auto-scrub on Disable as default | Currently configurable (default off); requires policy decision. | When stricter compliance regime is adopted. |
| MFA UX in BPP | Lives at IdP; no in-band MFA. | Never (by design — stays at IdP). |
| User preference panel beyond locale + avatar | Two-tier model is minimal. | Per product demand. |

### 7.2.2 Tenancy ([§4.2](04-bounded-contexts/4.2-tenancy.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Org sub-roles beyond Owner / Member | v1 uses one `OrganizationMember` role with `StoreAdminAssignment` as the only differentiator. | When Orgs need finance/billing/support specialists. |
| Sub-roles on Store Admin (e.g., catalog-only editor) | The matrix supports it; the entities don't (no sub-role per assignment). | When stores have distinct catalog vs. fulfillment staff. |
| Cross-Org migration of Stores | Stores are bound to one Org for life ([ADR-0018](../decisions/0018-organization-tenancy.md)). | Probably never; if needed, model as new Store + history archive. |
| Org-level Audit reads aggregated across stores | Audit access is per-store via assignment; Org-level read is achievable but not surfaced. | When operational dashboards demand cross-store views. |

### 7.2.3 Catalog ([§4.3](04-bounded-contexts/4.3-catalog.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Variant bundles (e.g., Family Combo S/M/L as native siblings of one parent) | Mutual-exclusion rule in v1 (per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md)) forbids variant × composite combinations. v1 workaround: three separate Composite Products optionally grouped via an additional Catalog. | When the workaround feels heavy for high-cardinality variant bundles. |
| Configurable variants (per-size extras menus) | Same mutual-exclusion rule. v1 workaround: one Configurable Product with size and extras as parallel `ChoiceGroup`s. | When the workaround proves UX-awkward at scale. |
| Nested composites (Composite inside Composite) | v1 mandates depth = 1. Workaround: pre-configure the inner composite as a Standalone Product. | When real-world composition exceeds one level. |
| `FIXED` choice group semantics (Beckn's spec-grey fourth `selectionType`) | Schema permits it but BAPs render inconsistently. Use `longDesc` for disclosure instead. | If Beckn ratifies semantics. |
| Daily capacity for Configurable Products ("I can make 30 custom cakes today") | Distinct concept from stock; would be a separate `DailyCapacity` entity. | When stores need service-window capacity controls. |
| Digital vs. physical product type taxonomy | v1 assumes physical. | When digital goods (downloads, codes) ship. |
| Full-text / faceted search implementation | Read-side concern; the domain is search-engine-agnostic. | When the platform needs Admin search beyond simple filters. |
| Media storage & CDN strategy | Domain knows only references; storage is operational. | Pre-launch — needed for actual deployment. |
| Cross-store product deduplication for discovery | Each store maintains independent Product entities; deduplication is a search-layer concern. | Discovery improvement; when CDS richer matching is needed. |
| Time-bound list-price changes (sale windows on catalog price) | Vouchers approximate; direct list-price scheduling is future. | When stores demand sale-pricing semantics. |
| Per-tenant categorization (store-private categories) | Categorization is platform-defined; stores use additional Catalogs instead. | Probably never — additional Catalogs serve this need. |
| Imperative bulk operations (import / migrate catalogs) | Operational tooling, not a domain feature. | Pre-launch onboarding of stores. |

### 7.2.4 Inventory ([§4.4](04-bounded-contexts/4.4-inventory.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Multi-location / multi-warehouse | v1 is single logical inventory per store. | When stores operate from multiple physical locations. |
| Backorders / preorders | Cannot sell beyond `stock_count` in v1. | When stores want to accept orders ahead of receipt. |
| Low-stock thresholds and notifications | Operational/UI feature; not domain. | Per product demand. |
| Automated supplier integrations / receiving | Stock receipt is manual today; `Received` events would arrive from an integration in future. | When suppliers integrate. |
| Inventory reconciliation flows (physical count vs. system) | Currently handled via `Corrected` events; UX is not designed. | Per operational maturity. |
| Cached availability projection | Synchronous queries are fine for v1 traffic; projection added when scale demands. | When `GetAvailability` traffic exceeds the synchronous budget. |
| Cross-store unified inventory | Each store's inventory is independent. | Probably never — would dissolve store boundaries. |

### 7.2.5 Promotion ([§4.5](04-bounded-contexts/4.5-promotion.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Platform-wide promotions | v1 is store-scoped only. Would introduce a `platform_promotion` discriminator. | When platform-level marketing campaigns ship. |
| Rule-based promotions (BOGO, X-for-Y, automatic cart discounts) | v1 is voucher-codes only. | When merchandising sophistication is needed. |
| Customer-specific pricing (B2B tiers, member rates) | No buyer-as-User in v1 ([ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)); buyer identity is opaque from BAPs. | If/when buyer identity flows through Beckn richly. |
| Voucher stacking (multiple per order) | v1 rule is one per order. | When merchants demand combinable discounts. |
| Voucher distribution mechanics (sharing codes by email/link) | Operational concern; not domain. | Per product demand. |
| Voucher exhaustion + extension UX | Operational flow; not domain. | Per product demand. |

### 7.2.6 Order & Fulfillment ([§4.6](04-bounded-contexts/4.6-order.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Refund / return workflows post-fulfillment | Cancel is pre-fulfillment only in v1; returns need new state + Inventory return path. | After payment-gateway integration. |
| Payment-gateway integration | Payment is fully external in v1; `payment_status` driven by external signals. | When BPP needs to settle payments directly (PCI scope). |
| Platform-managed logistics | Stores self-fulfill; logistics-provider integration is future. | When buyer demand or platform value-add requires it. |
| Multi-shipment / split orders | One-shot fulfillment in v1. | When line items can ship separately. |
| Subscriptions / recurring orders | Order is one-shot; subscriptions need new modeling. | When subscription commerce is launched. |
| Marketplace fees / payouts to stores | Settlement is external; BPP doesn't take a cut in v1. | When monetization is added. |
| Beckn `/track`, `/update`, `/rate`, `/support` handlers | Out of v1 transactional surface. | When ecosystem requires post-fulfillment buyer touchpoints. |
| ION extensions `/raise`, `/reconcile` (grievance / settlement) | Not part of v1 transactional flow. | When network adoption requires it. |
| Quote refresh / re-quote on TTL expiry | v1 expires the Order; client must restart. | When UX research demands smoother retry. |
| Partial cancel (cancel some line items, keep others) | v1 cancels the whole Order. | When use cases demand it. |

### 7.2.7 Audit ([§4.7](04-bounded-contexts/4.7-audit.md))

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Cryptographic chaining (each record references prior-hash) | Append-only at DB role level is sufficient for v1; chaining adds tamper-evidence beyond access control. | When compliance regime explicitly demands. |
| Higher-level audit-operations view (events grouped into business operations) | v1 is one-record-per-event; group-by-operation is a read-side projection. | When user-facing audit UX demands it. |
| Cross-tenant audit summarization | Analytics surface; not Audit's scope. | When platform-level reporting is built. |
| External SIEM / log shipping | Operational; can be added without domain change. | Per security-team requirements. |
| Audit-driven anomaly detection | Out of v1 scope. | When operational maturity warrants. |

---

## 7.3 Deferred cross-cutting concerns

### Consistency and orchestration

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Saga / process-manager framework | All v1 flows complete in seconds; hand-coded compensation per [§5.3.3](05-cross-cutting.md) suffices. | When long-running multi-context flows arrive (payment gateway, logistics, subscription billing). |
| Cross-aggregate event ordering | Per-aggregate ordering only in v1 ([§5.2.4](05-cross-cutting.md)); cross-aggregate sequencing is subscriber-side buffering. | When a use case demands it. |
| Distributed transactions (2PC, XA) | Forbidden by architecture ([§5.3.2](05-cross-cutting.md)). | Never (the architectural commitment is permanent). |

### Identity and authentication

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Multi-IdP linking (cross-references §7.2.1) | See above. | See above. |
| SCIM / IdP user provisioning push | JIT provisioning ([§4.1](04-bounded-contexts/4.1-identity.md)) is sufficient. | When enterprise customers demand. |
| Step-up authentication for sensitive actions | Lives at IdP; not in BPP. | If/when IdP supports it. |

### Localization

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Automatic translation | Bridge falls back to default locale on missing entries ([§5.7.6](05-cross-cutting.md)); no MT in v1. | Per product demand; would add a translation-port adapter. |
| Right-to-left script support specifics | Indonesian is LTR; no RTL UX testing in v1. | When markets expand. |
| Multi-currency display per locale | v1 currency is per-store and immutable; no per-buyer currency conversion. | When cross-currency support is needed. |
| Cross-locale uniqueness checks | Default-locale uniqueness only ([§5.7.5](05-cross-cutting.md)). | If duplicate-across-locale issues arise. |

### Data protection

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Field-level encryption beyond explicit fields | Baseline is TLS + DB-at-rest ([§5.6.9](05-cross-cutting.md)); field-level on demand only. | When compliance demands specific fields. |
| Multi-region data residency | Default is Indonesia ([§5.6.10](05-cross-cutting.md)). | When market expansion requires. |
| Cross-tenant analytics with consent flow | v1 aggregate-only; consent UX deferred. | When platform analytics products ship. |

---

## 7.4 Deferred operational capabilities

| Deferral | Why deferred | Likely v-when |
|---|---|---|
| Payment-gateway integration | Payment is external in v1 ([§4.6](04-bounded-contexts/4.6-order.md)). | When BPP needs to capture funds directly. |
| Logistics-provider integration | Self-fulfilled in v1. | When platform offers fulfillment services. |
| Bulk catalog import via API | Operational; UI-only in v1. | Pre-launch onboarding tooling. |
| External SIEM / observability integrations | Operational; the team picks the stack. | Per security/observability requirements. |
| Disaster-recovery drills / standby region | Operational; deferred to launch readiness. | Pre-launch. |
| Performance benchmarks at projected scale | The architecture is topology-neutral; scaling is operational. | When real traffic appears. |
| Automated PII-scrub scheduling beyond default-off | Available; default off. | When stricter regime is adopted. |
| Backup / restore runbook (per [§6.7](06-operational.md)) | Operational; needs platform decision. | Pre-launch. |

---

## 7.5 Known limits and assumptions

These are **not deferred features** — they're **constraints** that v1 lives within. Understanding them helps avoid mistaking a deliberate boundary for a missing feature.

### Buyer model

- **The BPP has no direct contact with buyers** ([ADR-0021](../decisions/0021-pure-bpp-no-storefront.md)). Buyers exist only as Beckn `transaction_id` + `bap_id` (with a `contact_snapshot` captured at `/init`). No `User`-typed buyer in v1.
- **Buyer authentication and lifecycle are owned by the BAP**, not the BPP.
- **Buyer-side analytics (re-purchase rates, lifetime value) are limited** to whatever the BAP exposes via Beckn fields.

### Currency

- **One currency per store**, immutable after first Active product ([§5.7.8](05-cross-cutting.md)).
- **No cross-currency conversion** between BPP and BAPs in v1.

### Tax

- **One `tax_rate` per Product**, applied to all variants ([§4.3](04-bounded-contexts/4.3-catalog.md)).
- **No multi-jurisdictional tax** (single jurisdiction per store in v1).
- **Tax inclusive vs. exclusive**: stored tax-excluded with derived `published_price`; receipts show both.

### Identity

- **Single linked identity per User** in v1.
- **Email verification is trusted from the IdP** (`email_verified` claim).
- **No password storage anywhere in the platform** — full IdP delegation.

### Concurrency / consistency

- **Strong within context, eventual across** ([§5.3](05-cross-cutting.md)). Cross-context state may lag by milliseconds.
- **No global transaction across contexts**, even when DB-physical sharing would allow it ([§5.3.2](05-cross-cutting.md)).
- **At-least-once event delivery** ([§5.2.3](05-cross-cutting.md)). Subscribers must dedup.

### Order

- **Quote TTL default 15 minutes** ([§4.6](04-bounded-contexts/4.6-order.md)). Catalog price changes during the TTL don't propagate; snapshots are sticky.
- **Pre-fulfillment cancel only**. No mid-fulfillment cancel; no post-fulfillment refund/return path.

### Inventory

- **Single logical inventory per store**. No location split.
- **No oversell**: `purchasable` + `stock_count` strictly gate purchases.

### Catalog

- **Default Catalog membership is mandatory** ([ADR-0020](../decisions/0020-default-catalog-mandatory.md)). To hide a Product from Beckn, set `status = Archived` or `Draft`.
- **A Product's `mode` (Standalone / Variant / Configurable / Composite) is immutable** after creation (per [ADR-0023](../decisions/0023-product-composition-and-choice-modeling.md)). Modes are mutually exclusive — no variant bundles, no configurable variants, no nested composites in v1.

### Audit

- **Records cannot be modified** except via the narrow `audit_pii_scrubber` role acting on `source_envelope` ([§4.7](04-bounded-contexts/4.7-audit.md)).
- **Retention is per category**, not per record; the categorization is part of the operational config.

### Localization

- **Platform default locale is `id`** (Bahasa Indonesia). Every `LocalizedText` must contain an `id` entry.
- **No automatic translation**: missing locales fall back to default.

---

## 7.6 Migration paths for deferred features

A few of the deferrals above will likely land first. Sketches of how to add them without churning the architecture:

### Adding multi-IdP linking

1. Introduce `ExternalIdentity` entity: `(id, user_id, external_subject_id, idp_name, linked_at)`.
2. `User.external_subject_id` becomes a derived/cached field of the **primary** ExternalIdentity (compat shim).
3. JIT provisioning matches by `(idp_name, external_subject_id)`; falls back to email match for linking flows (manual confirm).
4. New use case `LinkExternalIdentity(user, idp_name, sub)`.
5. ADR superseding [ADR-0009 §"single linked identity"](../decisions/0009-identity-and-external-idp.md).

### Adding payment-gateway integration

1. New Payment context with `PaymentIntent` entity; references Order by ID.
2. New port `PaymentGatewayPort` in Order; called at `/confirm` to create/authorize/capture.
3. Gateway webhook adapter drives `Order.payment_status` (replacing the external-signal model).
4. New ADRs for: payment context boundaries, PCI-scope minimization, refund flow, webhook idempotency.
5. Refund flow becomes the foundation for returns ([§7.2.6](#726-order--fulfillment-46)).

### Adding return / refund workflow

1. After payment gateway lands, add `Returned` state to Order; add Return entity tracking line items and reason.
2. Inventory `Returned` event already exists ([§4.4](04-bounded-contexts/4.4-inventory.md)) — wire it up.
3. New ADRs for the refund-vs-return distinction (financial vs. fulfillment).

### Adding platform-wide promotions

1. Voucher gains an optional `platform_id` field (nullable; if set, replaces `store_id`).
2. New System Admin capabilities: `platform_voucher.create`, `_disable`.
3. `Promotion.ValidateVoucher` adds a platform-vs-store check.
4. ADR superseding the "store-scoped only" rule in [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md).

### Adding multi-location inventory

1. New `Location` entity in Inventory.
2. `StockLevel` gains `location_id`; one `StockLevel` per `(inventory_item, location)` instead of per item.
3. Reservation gains `location_id`; allocation logic picks a location.
4. New ADRs for routing rules (closest? lowest stock?), cross-location transfers, and Beckn projection (does the BAP see one provider or many?).

### Adding sagas / process managers

1. New `ProcessInstance` entity per long-running flow; tracks current step and compensation pointers.
2. Each step is a use case; compensation is declared per step.
3. Recover-on-restart logic walks `ProcessInstance` table.
4. Existing hand-coded orchestrations (in Order) become candidates for migration to the saga framework.

---

> **Next**: [§8 References](08-references.md) — ADR index, design/* index, external references.
