# Design Gaps & Ambiguities

Unresolved questions in `CLAUDE.md`. Each file is a self-contained gap to be resolved one at a time. The numbering reflects a *suggested* resolution order based on dependencies — pick whichever you want next.

A gap is "resolved" when its decisions are folded into `CLAUDE.md` (or a sibling design document) and this file is moved to `gaps/resolved/`.

## Resolved

- **01 — Identity & Access scope** — resolved by [ADR-0009](../decisions/0009-identity-and-external-idp.md); full model in [design/identity.md](../design/identity.md); archived at [gaps/resolved/01-identity-and-access-scope.md](resolved/01-identity-and-access-scope.md).
- **02 — Platform-scoped roles** — resolved by [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md); archived at [gaps/resolved/02-platform-scoped-roles.md](resolved/02-platform-scoped-roles.md).
- **04 — Store lifecycle beyond creation** — resolved by [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md); archived at [gaps/resolved/04-store-lifecycle.md](resolved/04-store-lifecycle.md).
- **05 — BPP network identity** — resolved by [ADR-0001](../decisions/0001-bpp-network-identity.md); archived at [gaps/resolved/05-bpp-network-identity.md](resolved/05-bpp-network-identity.md).
- **06 — Store-to-Beckn publication** — resolved by [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md) and [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md); archived at [gaps/resolved/06-store-to-beckn-publication.md](resolved/06-store-to-beckn-publication.md).
- **07 — Catalog modeling scope** — resolved by [ADR-0004](../decisions/0004-store-publication-and-multi-catalog-projection.md) and [ADR-0005](../decisions/0005-catalog-and-product-modeling.md); full model in [design/catalog.md](../design/catalog.md); archived at [gaps/resolved/07-catalog-modeling-scope.md](resolved/07-catalog-modeling-scope.md).
- **08 — Inventory boundary** — resolved by [ADR-0006](../decisions/0006-inventory-model.md); archived at [gaps/resolved/08-inventory-boundary.md](resolved/08-inventory-boundary.md).
- **09 — Pricing & promotions** — resolved by [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md). Introduced the **Promotion** bounded context. Archived at [gaps/resolved/09-pricing-and-promotions.md](resolved/09-pricing-and-promotions.md).
- **10 — Localization & currency** — currency resolved by [ADR-0007](../decisions/0007-pricing-tax-and-vouchers.md); localization resolved by [ADR-0008](../decisions/0008-localization-and-localizedtext.md). Introduced the `LocalizedText` value object. Archived at [gaps/resolved/10-localization-and-currency.md](resolved/10-localization-and-currency.md).
- **12 — Invitation–account reconciliation** — resolved by [ADR-0010](../decisions/0010-invitation-account-reconciliation.md); archived at [gaps/resolved/12-invitation-account-reconciliation.md](resolved/12-invitation-account-reconciliation.md).
- **13 — Domain events design** — resolved by [ADR-0011](../decisions/0011-domain-events.md); full registry in [design/events.md](../design/events.md); archived at [gaps/resolved/13-domain-events-design.md](resolved/13-domain-events-design.md).

## Foundations

3. [Authorization & capability model](03-authorization-and-capabilities.md) *(partially resolved by ADR-0002)*

## Beckn integration shape

7. [Order & fulfillment phasing](11-order-and-fulfillment-phasing.md)

## Cross-cutting model concerns

14. [Cross-context consistency](14-cross-context-consistency.md)
15. [First-party idempotency](15-first-party-idempotency.md)
16. [Soft delete & audit log](16-soft-delete-and-audit.md)
17. [PII & compliance stance](17-pii-and-compliance.md)
