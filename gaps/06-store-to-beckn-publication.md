# Gap 06 — Store-to-Beckn publication

> **PARTIALLY RESOLVED on 2026-05-28** by [ADR-0003](../decisions/0003-store-lifecycle-and-state-machine.md). Closed: question 1 (default visibility — Draft is not published), question 2 (activation flow — platform-driven, criteria still open), question 4 (modeling location — publication is a Store-state concern at the store level), question 5 (revocation semantics — in-flight orders complete; no new orders), question 6 (lifecycle linkage — automatic via `StoreStatusChanged` event). Still open: question 3 (per-product / per-category visibility within an Active store), question 7 (compliance gating / activation checklist).

## Statement

The charter implies stores reach Beckn but does not specify the model by which a store becomes (or stops being) visible on the network. Is publication automatic, opt-in, or curated? Is it all-or-nothing per store, or selective per product/category? Where does the publication state live? This decision sits at the seam between the domain and the Bridge.

## Current charter coverage

* §2.1 — Platform "exposes stores to the Beckn network."
* §2.8 — "It must be possible to operate every store function… without any Beckn participant in the loop" — implies Beckn exposure is not a precondition for store operation.
* No discussion of how publication is triggered, gated, or revoked.

## Open questions

1. **Default visibility.** On store creation, is the store immediately searchable on Beckn, or off by default?
2. **Activation flow.** If opt-in: is publication a one-step toggle by the owner, or does it require platform review/approval?
3. **Granularity.** Is publication store-level only, or can individual products / categories be marked Beckn-visible vs. first-party-only?
4. **Modeling location.** Is publication state a Store attribute (Tenancy), a Catalog attribute (per product), or its own subdomain ("Beckn Presence")?
5. **Revocation semantics.** When publication is turned off, what happens to in-flight Beckn transactions? Cleanup vs. let-complete?
6. **Lifecycle linkage.** How does store suspension/archival (Gap 04) interact with publication? Is depublication automatic on suspension?
7. **Compliance gating.** Is there a checklist a store must satisfy before being publishable (verified contact, return policy, KYC, certifications)?

## Implications

* The Bridge's `on_search` response depends on knowing which stores and items to surface.
* Without an opt-in flow, every new store appears on Beckn immediately — likely undesirable from a quality perspective.
* The publication model affects how the network registry sees the platform.

## Dependencies

* Depends on: 04 (store lifecycle), 05 (BPP identity).
* Influences: 07 (catalog visibility flags), Bridge implementation.
