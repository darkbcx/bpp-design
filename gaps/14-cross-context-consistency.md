# Gap 14 — Cross-context consistency

## Statement

§2.6 lists two inter-context channels (synchronous queries/commands and domain events) but does not state which is the default, what consistency guarantees each provides, or how multi-context operations are coordinated. The system's overall consistency stance — strongly consistent across contexts vs. eventually consistent — is unspecified.

## Current charter coverage

* §2.6 — Two channels; choice "documented at the time it is made."
* §2.6 — Identifiers crossing contexts are "opaque references, not foreign keys."
* §3.3 — Aggregates must be modified together transactionally; cross-aggregate is by reference.
* No stance on cross-context transactions, sagas, or eventual consistency.

## Open questions

1. **Default consistency.** Is the system strongly consistent across contexts at launch (single database, single transaction can span contexts) or eventually consistent from the start (each context's data is updated independently)?
2. **Transactional scope.** Can a single Application use case write to multiple contexts atomically, or is each cross-context call a separate transaction?
3. **Sagas / process managers.** For multi-step flows (e.g., accept invitation → create membership → notify owner), is there an explicit saga concept, or are these implemented ad hoc?
4. **Read consistency.** When the storefront reads a product (Catalog) and its stock (Inventory), is a stale read acceptable? How stale?
5. **Failure handling.** Event published, but downstream subscriber fails. Is there a retry mechanism, dead-letter queue, manual reconciliation?
6. **Outbox / inbox patterns.** If events must be reliable, is the transactional outbox pattern adopted, or something else?
7. **Future-proofing for service split.** If today contexts share a database, will future extraction be a refactor or a rewrite?

## Implications

* The Bridge's order flow (Gap 11) crosses Inventory, Catalog, and Order. The consistency choice here determines how that flow is built.
* "No foreign keys across contexts" (§2.6) suggests an eventual stance, but the charter doesn't commit.
* Affects testing strategy and operational complexity.

## Dependencies

* Depends on: 13 (events).
* Influences: 11 (order flows), 15 (idempotency).
