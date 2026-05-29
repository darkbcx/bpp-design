# Design Gaps & Ambiguities

All originally identified gaps have been resolved. This file is kept as the historical index.

## Resolved

- **01 — Identity & Access scope** — resolved by [ADR-0009](../decisions/0009-identity-and-external-idp.md); full model in [design/identity.md](../design/identity.md); archived at [gaps/resolved/01-identity-and-access-scope.md](resolved/01-identity-and-access-scope.md).
- **02 — Platform-scoped roles** — resolved by [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md); archived at [gaps/resolved/02-platform-scoped-roles.md](resolved/02-platform-scoped-roles.md).
- **03 — Authorization & capability model** — resolved by [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md) and [ADR-0016](../decisions/0016-authorization-details.md); archived at [gaps/resolved/03-authorization-and-capabilities.md](resolved/03-authorization-and-capabilities.md).
- **04 — Store lifecycle beyond creation** — resolved by [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md); archived at [gaps/resolved/04-store-lifecycle.md](resolved/04-store-lifecycle.md).
- **05 — BPP network identity** — resolved by [ADR-0001](../decisions/0001-bpp-network-identity.md); archived at [gaps/resolved/05-bpp-network-identity.md](resolved/05-bpp-network-identity.md).
- **06 — Store-to-Beckn publication** — resolved by [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md) and [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md); archived at [gaps/resolved/06-store-to-beckn-publication.md](resolved/06-store-to-beckn-publication.md).
- **07 — Catalog modeling scope** — resolved by [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) and [ADR-0005](../decisions/0005-catalog-and-product-modeling.md); full model in [design/catalog.md](../design/catalog.md); archived at [gaps/resolved/07-catalog-modeling-scope.md](resolved/07-catalog-modeling-scope.md).
- **08 — Inventory boundary** — resolved by [ADR-0006](../decisions/0006-inventory-model.md); archived at [gaps/resolved/08-inventory-boundary.md](resolved/08-inventory-boundary.md).
- **09 — Pricing & promotions** — resolved by [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md). Introduced the **Promotion** bounded context. Archived at [gaps/resolved/09-pricing-and-promotions.md](resolved/09-pricing-and-promotions.md).
- **10 — Localization & currency** — currency resolved by [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md); localization resolved by [ADR-0008](../decisions/0008-localization-and-localizedtext.md). Introduced the `LocalizedText` value object. Archived at [gaps/resolved/10-localization-and-currency.md](resolved/10-localization-and-currency.md).
- **11 — Order & fulfillment phasing** — resolved by [ADR-0017](../decisions/0017-order-and-fulfillment.md); full Order context model in [design/order.md](../design/order.md). Introduced the **Order & Fulfillment** bounded context. Archived at [gaps/resolved/11-order-and-fulfillment-phasing.md](resolved/11-order-and-fulfillment-phasing.md).
- **12 — Invitation–account reconciliation** — resolved by [ADR-0010](../decisions/0010-invitation-account-reconciliation.md); archived at [gaps/resolved/12-invitation-account-reconciliation.md](resolved/12-invitation-account-reconciliation.md).
- **13 — Domain events design** — resolved by [ADR-0011](../decisions/0011-domain-events.md); full registry in [design/events.md](../design/events.md); archived at [gaps/resolved/13-domain-events-design.md](resolved/13-domain-events-design.md).
- **14 — Cross-context consistency** — resolved by [ADR-0012](../decisions/0012-cross-context-consistency.md); archived at [gaps/resolved/14-cross-context-consistency.md](resolved/14-cross-context-consistency.md).
- **15 — First-party idempotency** — resolved by [ADR-0013](../decisions/0013-first-party-idempotency.md); archived at [gaps/resolved/15-first-party-idempotency.md](resolved/15-first-party-idempotency.md).
- **16 — Soft delete & audit log** — resolved by [ADR-0014](../decisions/0014-soft-delete-and-audit.md); full Audit context model in [design/audit.md](../design/audit.md). Introduced the **Audit** bounded context. Archived at [gaps/resolved/16-soft-delete-and-audit.md](resolved/16-soft-delete-and-audit.md).
- **17 — PII & compliance stance** — resolved by [ADR-0015](../decisions/0015-pii-and-right-to-erasure.md); full PII catalog in [design/pii.md](../design/pii.md); archived at [gaps/resolved/17-pii-and-compliance.md](resolved/17-pii-and-compliance.md).

The design is complete. Next step: consolidate into `DESIGN.md` for handoff (CLAUDE.md §10.3).
