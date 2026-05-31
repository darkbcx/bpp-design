# 8. References

Index of every authoritative artifact the handoff package draws from. Use this to find the **historical reasoning** behind a rule, the **full entity model** behind a context summary, or the **external standard** behind a protocol detail.

| # | Topic |
|---|---|
| 8.1 | Architecture Decision Records (ADRs) |
| 8.2 | Sub-design documents |
| 8.3 | Charter and meta-documents |
| 8.4 | External standards and protocols |
| 8.5 | Compliance and legal references |
| 8.6 | Reference repositories |

---

## 8.1 Architecture Decision Records

The ADRs in [`/decisions/`](../decisions/) are the **historical record** of every significant judgment call. They are **immutable once accepted**; a superseded ADR is replaced by a new one rather than rewritten ([§10.4 of CLAUDE.md](../CLAUDE.md)).

When the handoff and an ADR seem to disagree: the **ADR wins on decision content** (what was decided, what was rejected, why); the **handoff wins on current vocabulary and integration** (since wire vocabulary and terminology have evolved across ADRs).

### Active ADRs (chronological)

| # | Title | Topic in handoff |
|---|---|---|
| [ADR-0001](../decisions/0001-bpp-network-identity.md) | BPP network identity | Platform is one BPP; stores project as `Provider` nodes ([§3](03-beckn-integration.md)) — signing / registry / callback parts superseded by ADR-0022 |
| [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) | Authorization tiers and matrix | Three tiers, capability matrix, active store ([§5.1](05-cross-cutting.md)) |
| [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md) | Store lifecycle and state machine | Draft / Active / Suspended / Paused ([§4.2](04-bounded-contexts/4.2-tenancy.md)) |
| [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) | Store publication and multi-catalog projection | Multi-catalog on Beckn ([§3.4.1](03-beckn-integration.md), [§4.3](04-bounded-contexts/4.3-catalog.md)) |
| [ADR-0005](../decisions/0005-catalog-and-product-modeling.md) | Catalog and product modeling | Matrix vs. Flat product modes ([§4.3](04-bounded-contexts/4.3-catalog.md)) |
| [ADR-0006](../decisions/0006-inventory-model.md) | Inventory model | StockLevel + Reservation + StockMovement ([§4.4](04-bounded-contexts/4.4-inventory.md)) |
| [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md) | Pricing, tax, and vouchers | Money, base_price + tax_rate, store-scoped vouchers ([§4.3](04-bounded-contexts/4.3-catalog.md), [§4.5](04-bounded-contexts/4.5-promotion.md)) |
| [ADR-0008](../decisions/0008-localization-and-localizedtext.md) | Localization and LocalizedText | Value object; default locale `id` ([§5.7](05-cross-cutting.md)) |
| [ADR-0009](../decisions/0009-identity-and-external-idp.md) | Identity and external IdP | OIDC integration, two-tier User profile ([§4.1](04-bounded-contexts/4.1-identity.md)) |
| [ADR-0010](../decisions/0010-invitation-account-reconciliation.md) | Invitation and account reconciliation | Email-targeted invitations, IdP-verified acceptance ([§4.2](04-bounded-contexts/4.2-tenancy.md)) |
| [ADR-0011](../decisions/0011-domain-events.md) | Domain events | Transactional outbox + envelope schema ([§5.2](05-cross-cutting.md)) |
| [ADR-0012](../decisions/0012-cross-context-consistency.md) | Cross-context consistency | Strong-within / eventual-across; orchestration with compensation ([§5.3](05-cross-cutting.md)) |
| [ADR-0013](../decisions/0013-first-party-idempotency.md) | First-party idempotency | Client UUID keys ([§5.4](05-cross-cutting.md)) |
| [ADR-0014](../decisions/0014-soft-delete-and-audit.md) | Soft-delete pattern and the Audit context | Business vs. operational entities; Audit context ([§4.7](04-bounded-contexts/4.7-audit.md), [§5.5](05-cross-cutting.md)) |
| [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md) | PII handling and right-to-erasure | Scrub in place; PII catalog ([§5.6](05-cross-cutting.md)) |
| [ADR-0016](../decisions/0016-authorization-details.md) | Authorization details | Action-level naming; requireCapability pattern ([§5.1](05-cross-cutting.md)) |
| [ADR-0017](../decisions/0017-order-and-fulfillment.md) | Order and Fulfillment | Order state machine, CDS publish, Beckn v2 transactional flow ([§4.6](04-bounded-contexts/4.6-order.md)) — Buyer / Order.Place parts superseded by ADR-0021; direct-CDS-publish parts superseded by ADR-0022 |
| [ADR-0018](../decisions/0018-organization-tenancy.md) | Organization tenancy | Org as top-level tenant grouping stores ([§4.2](04-bounded-contexts/4.2-tenancy.md)) |
| [ADR-0019](../decisions/0019-manual-catalog-republication.md) | Manual catalog republication | `Store.RequestRepublish` + `Org.RequestRepublishAll` ([§4.2](04-bounded-contexts/4.2-tenancy.md)) |
| [ADR-0020](../decisions/0020-default-catalog-mandatory.md) | Default Catalog mandatory membership | Every Product is in Default; hide via Product status ([§4.3](04-bounded-contexts/4.3-catalog.md)) — supersedes parts of ADR-0004 |
| [ADR-0021](../decisions/0021-pure-bpp-no-storefront.md) | Pure BPP, no first-party storefront | User scope is tenant/platform only; Order.buyer is Beckn-typed ([§1](01-overview.md), [§4.1](04-bounded-contexts/4.1-identity.md), [§4.6](04-bounded-contexts/4.6-order.md)) — supersedes parts of ADR-0009 and ADR-0017 |
| [ADR-0022](../decisions/0022-onix-protocol-gateway.md) | ONIX as Beckn protocol gateway | ONIX (vendor binary, per-BPP) handles signing / schema validation / registry / CDS publish. BPP keeps domain ↔ wire mapping, inbound re-verification, ION-XXXX error mapping, Ack/Nack with CounterSignature ([§3](03-beckn-integration.md)) — supersedes parts of ADR-0001 and ADR-0017 |

### How ADRs supersede each other

| Superseded portion | By |
|---|---|
| ADR-0004 §"Default Catalog opt-out" | ADR-0020 (mandatory membership) |
| ADR-0009 §"User as buyer-or-tenant" | ADR-0021 (User is tenant/platform only) |
| ADR-0017 §"polymorphic Buyer" + §"first-party Order entry" | ADR-0021 (Beckn-typed Buyer; no first-party entry) |
| ADR-0001 §"signing key custody" + §"registry interactions" + §"Beckn callback endpoint exposure" | ADR-0022 (ONIX as protocol gateway) |
| ADR-0017 §"direct CDS publishing from the Bridge" | ADR-0022 (BPP publishes to ONIX; ONIX forwards to CDS) |

Superseded ADRs remain in the repository for historical traceability. Their `Status:` field is updated to point at the superseding ADR.

### Template

[`decisions/0000-template.md`](../decisions/0000-template.md) — used for new ADRs.

---

## 8.2 Sub-design documents

The design docs in [`/design/`](../design/) carry the **full entity models, lifecycles, and use-case catalogs** that the handoff §4 sections summarize. When you need the schema-level detail, the design doc is the source.

| Doc | Backed by | Covers |
|---|---|---|
| [`design/identity.md`](../design/identity.md) | ADR-0009, ADR-0021 | User and Session entities; JIT provisioning; IdP integration boundary |
| [`design/tenancy.md`](../design/tenancy.md) | ADR-0018, ADR-0003, ADR-0010 | Organization, OrganizationMember, OrganizationInvitation, Store, StoreAdminAssignment; full lifecycle tables |
| [`design/catalog.md`](../design/catalog.md) | ADR-0005, ADR-0004, ADR-0007, ADR-0008, ADR-0020 | Product, ProductVariant, ProductAttribute, Catalog, CatalogMembership, PlatformCategory, ProductCategoryAssignment, Media; full use-case catalog |
| [`design/order.md`](../design/order.md) | ADR-0017, ADR-0021 | Order, Quote, LineItem; state machine; orchestration pseudocode; Bridge ↔ Order mapping table; sequence walkthroughs |
| [`design/events.md`](../design/events.md) | ADR-0011 | Authoritative event registry (~70 declared event types); envelope schema; subscriber registry |
| [`design/audit.md`](../design/audit.md) | ADR-0014, ADR-0015 | AuditRecord schema; retention table per category; access matrix; DB role separation |
| [`design/pii.md`](../design/pii.md) | ADR-0015 | PII field catalog (per entity, per event payload); scrub-action types; processor catalog |

### Template

[`design/0000-template.md`](../design/0000-template.md) — used for new design docs.

---

## 8.3 Charter and meta-documents

| Doc | Role |
|---|---|
| [`/CLAUDE.md`](../CLAUDE.md) | **The architectural charter.** Principles, rules, boundaries. The binding document for design-time decisions. |
| [`/handoff/`](README.md) (this directory) | **The handoff reading view** — the integrated design package this index belongs to. |
| [`/gaps/`](../gaps/) | Open design questions awaiting resolution. Most are now resolved; resolved gaps are in `gaps/resolved/` with pointers to the ADRs that resolved them. |
| [`/gaps/resolved/`](../gaps/resolved/) | Historical record of gaps and which ADRs resolved them. |

---

## 8.4 External standards and protocols

### Beckn / ION

The protocol the BPP integrates with. Read these when you're working in the Bridge ([§3](03-beckn-integration.md), [§4.6](04-bounded-contexts/4.6-order.md)) or implementing the wire mapping registry.

| Reference | Description |
|---|---|
| **ION specifications** | Locally vendored at `/Users/danielignatius/mydev/personal/ion-specs` (per [§7 of CLAUDE.md](../CLAUDE.md)) — source of truth for Beckn v2 / ION schemas, message structures, examples. |
| **Beckn Protocol v2** | Field names, envelope structure, error codes. Referenced by the Bridge's mapping registry. |
| **Beckn Network specifications** | Registry semantics, participant identity (`bpp-id`, `bpp-uri`), signing requirements, callback semantics. Referenced by [ADR-0001](../decisions/0001-bpp-network-identity.md). |
| **CDS (Catalog Discovery Service)** | The `/catalog/publish` endpoint pattern is referenced by [ADR-0017](../decisions/0017-order-and-fulfillment.md) and the Bridge subscriber config. |

### Identity

| Reference | Description |
|---|---|
| **OpenID Connect (OIDC) Core 1.0** | Authentication protocol the BPP integrates with. Referenced by [ADR-0009](../decisions/0009-identity-and-external-idp.md). Specifies the claims (`sub`, `email`, `email_verified`, `name`, `picture`, `locale`) the BPP consumes. |
| **OAuth 2.0** (RFC 6749) | OIDC's underlying authorization framework. |
| **JWT / JWS / JWK** (RFC 7515 / 7517 / 7519) | Token formats used by the IdP. Implementation-side concern; not surfaced in the domain. |

### Internationalization

| Reference | Description |
|---|---|
| **BCP 47 / RFC 5646** | Language tag format used throughout `LocalizedText` ([§5.7](05-cross-cutting.md)). Examples: `id`, `en`, `en-ID`, `ms`, `jv`. |
| **ISO 639-1 / 639-3** | Language codes underpinning BCP 47. |
| **CLDR (Common Locale Data Repository)** | For locale data beyond what BCP 47 specifies (number formatting, dates) — operational, not domain. |

### Money and currency

| Reference | Description |
|---|---|
| **ISO 4217** | Currency code format used by the `Money` value object ([§5.7.8](05-cross-cutting.md), [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md)). `IDR`, `USD`, `MYR`, etc. |
| **Minor units** | Each ISO 4217 currency declares its minor-unit divisor. `Money` stores integer minor units. |

### Cryptography / transport

| Reference | Description |
|---|---|
| **TLS 1.2+** | Transport encryption baseline ([§5.6.9](05-cross-cutting.md)). |
| **Ed25519 / RSA** | Beckn signing algorithms (per Beckn spec). Referenced by [ADR-0001](../decisions/0001-bpp-network-identity.md). |

---

## 8.5 Compliance and legal references

| Reference | Description |
|---|---|
| **Indonesia PDP law** (Undang-Undang Pelindungan Data Pribadi, UU 27/2022) | Primary compliance regime for v1 ([§5.6](05-cross-cutting.md), [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md)). Data-subject rights, residency, breach notification. |
| **GDPR-compatible design** | The PII scrubbing and right-to-erasure model is GDPR-compatible by design; specific GDPR adoption is not in v1 scope. |
| **PCI DSS** | Payment Card Industry Data Security Standard — explicitly out of scope for v1 (no card data in the BPP per [§4.6](04-bounded-contexts/4.6-order.md)). Would re-enter scope with payment-gateway integration ([§7.6](07-open-issues.md)). |
| **OWASP Top 10** | Application security baseline ([§8 of CLAUDE.md](../CLAUDE.md)). Implementation guidance, not domain concern. |

---

## 8.6 Reference repositories

These are **inspiration / consultation references**, not parts of this system. Treat them per [§7 of CLAUDE.md](../CLAUDE.md).

| Repository | Role | How to use it |
|---|---|---|
| `/Users/danielignatius/mydev/personal/ion-specs` | Beckn / ION protocol source-of-truth | Look up field names, structures, examples. **Used only in the Bridge.** |
| `/Users/danielignatius/mydev/personal/bitemycart` | Reference for ecommerce *shape* | Learn what problems exist; design solutions for our multi-tenant context from first principles. **Do not copy schema, file structure, or framework choices.** |

---

## How to find the right reference

| You're looking for… | Go to |
|---|---|
| The **rule** | The relevant `handoff/` section + cross-reference to ADR |
| The **reason behind the rule** | The ADR linked from the rule |
| **Full entity schemas** | The relevant `design/*.md` doc |
| **A protocol field's exact name / shape** | `ion-specs` (Bridge work only) |
| **The list of all events** | [`design/events.md`](../design/events.md) |
| **The list of all PII fields** | [`design/pii.md`](../design/pii.md) |
| **The capability catalog** | [§5.1.2](05-cross-cutting.md) (handoff) + per-context use-case lists in [§4](04-bounded-contexts/README.md) |
| **What was deferred and why** | [§7](07-open-issues.md) |
| **How to operate the system** | [§6](06-operational.md) |

---

> **End of handoff package.** From here, the implementing team is the source of truth.
