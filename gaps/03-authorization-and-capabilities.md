# Gap 03 — Authorization & capability model

> **PARTIALLY RESOLVED on 2026-05-28** by [ADR-0002](../decisions/0002-authorization-tiers-and-matrix.md). Closed: questions 2 (catalog ownership — system-defined, Tenancy-owned), 3 (matrix mutability — runtime by System Admin only, not per-store), and 4 (composition with platform scope). Question 1 (granularity) is partly answered (capabilities map to concrete actions, action-level). Still open: question 5 (decision-exposure pattern), question 6 (negative permissions), question 7 (audit of denied attempts — see Gap 16).

## Statement

The charter says authorization is capability-based and store-scoped (§5.3, §6.5), and that a documented role→capability matrix exists in the Tenancy context. But the matrix itself, the capability catalog, the granularity, and the composition with platform roles are all undefined. "Capability" is invoked as a primitive without being specified.

## Current charter coverage

* §5.3 — "Capabilities derive from role within store context. Authorization questions are always of the form: 'Does this User have this capability in this Store?'"
* §6.5 — "Capabilities map to roles via a documented matrix. The matrix is owned by the Tenancy context."
* §6.5 — "Adding roles later is a deliberate design action — a new role must be justified by a distinct set of capabilities."

## Open questions

1. **Capability granularity.** Are capabilities action-level (`product.create`, `product.publish`, `member.invite`) or resource-level (`manage_catalog`, `manage_members`)? Mixed?
2. **Capability catalog ownership.** Is the capability catalog hard-coded, config-driven, or stored? Can it be extended without code changes?
3. **Matrix mutability.** Are role→capability mappings fixed for all stores, or can a store owner customize them (e.g., create a "Manager" role with a subset)?
4. **Composition with platform scope.** How does a platform admin's authority compose with store-scoped capabilities? Does a platform role implicitly grant all store capabilities, or only specific cross-tenant ones?
5. **Decision exposure.** How is an authorization decision surfaced to the Application Layer — predicate functions, policy objects, an `Authorization` service, or attribute-based access control (ABAC)?
6. **Negative permissions.** Are there explicit denies, or is the model purely additive?
7. **Audit of authorization failures.** Are denied attempts logged? Where?

## Implications

* Every Application use case must perform authorization before acting. The shape of that check is a recurring code pattern; getting it wrong is a tenancy-leak risk.
* Adding new features (e.g., promotions, payouts) will continually require new capabilities; the model must accommodate growth without per-feature special cases.
* Influences how the Bridge invokes use cases on behalf of remote Beckn actors — the Bridge needs a defensible "actor" to attach to each call.

## Dependencies

* Depends on: 01 (User), 02 (Platform roles).
* Blocks: most Application Layer design.
