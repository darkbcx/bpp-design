# Design Gaps & Ambiguities

Unresolved questions in `CLAUDE.md`. Each file is a self-contained gap to be resolved one at a time. The numbering reflects a *suggested* resolution order based on dependencies — pick whichever you want next.

A gap is "resolved" when its decisions are folded into `CLAUDE.md` (or a sibling design document) and this file is moved to `gaps/resolved/`.

## Resolved

- **02 — Platform-scoped roles** — resolved by [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md); archived at [gaps/resolved/02-platform-scoped-roles.md](resolved/02-platform-scoped-roles.md).
- **04 — Store lifecycle beyond creation** — resolved by [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md); archived at [gaps/resolved/04-store-lifecycle.md](resolved/04-store-lifecycle.md).
- **05 — BPP network identity** — resolved by [ADR-0001](../decisions/0001-bpp-network-identity.md); archived at [gaps/resolved/05-bpp-network-identity.md](resolved/05-bpp-network-identity.md).

## Foundations

1. [Identity & Access scope](01-identity-and-access-scope.md)
3. [Authorization & capability model](03-authorization-and-capabilities.md) *(partially resolved by ADR-0002)*

## Beckn integration shape

6. [Store-to-Beckn publication](06-store-to-beckn-publication.md) *(partially resolved by ADR-0003)*
7. [Order & fulfillment phasing](11-order-and-fulfillment-phasing.md)

## Catalog & commerce

8. [Catalog modeling scope](07-catalog-modeling-scope.md)
9. [Inventory boundary](08-inventory-boundary.md)
10. [Pricing & promotions](09-pricing-and-promotions.md)
11. [Localization & currency](10-localization-and-currency.md)

## Cross-cutting model concerns

12. [Invitation–account reconciliation](12-invitation-account-reconciliation.md)
13. [Domain events design](13-domain-events-design.md)
14. [Cross-context consistency](14-cross-context-consistency.md)
15. [First-party idempotency](15-first-party-idempotency.md)
16. [Soft delete & audit log](16-soft-delete-and-audit.md)
17. [PII & compliance stance](17-pii-and-compliance.md)
