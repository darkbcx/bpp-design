# Gap 11 — Order & fulfillment phasing

## Statement

The charter lists Order & Fulfillment as a bounded context but marks it "Out of scope for the initial phase, but reserved here so it does not collide with other contexts later" (§2.5). Meanwhile, the Beckn protocol's core flows (`search` → `select` → `init` → `confirm` → `status` → `update`) are *fundamentally* order-centric. A Beckn integration that only handles `search` is a partial implementation. There is a tension between the stated phasing and the protocol's shape.

## Current charter coverage

* §2.5 — Order & Fulfillment reserved as a context, scope deferred.
* §4 — Beckn Bridge is fully designed but its inbound `select`/`init`/`confirm` handlers have no domain target to call.

## Open questions

1. **Minimum viable Beckn surface.** Can a first release ship with `search`/`on_search` only, with `select`/`init`/`confirm` returning a clean "not supported" Beckn error? Is that acceptable to the network?
2. **Minimum order domain.** What is the smallest order model that makes end-to-end Beckn work — an immutable order header + line items, no fulfillment workflow?
3. **Stateful "select".** Beckn `select` returns a quote that may or may not persist. Is the quote a first-class entity, a cached projection, or a stateless computation?
4. **Init vs. confirm.** `init` reserves; `confirm` commits. What does each map to in the domain — Order in different states, or two entities (Quote, Order)?
5. **Payment integration.** Are payments external to the BPP (handled by buyer-side or network-mandated rails), or do we accept payment? Stance affects scope dramatically.
6. **Fulfillment.** Self-fulfillment by stores, platform-managed logistics, or fully external? Beckn's `fulfillment` object expects answers.
7. **Cancellation / returns.** In or out of phase one?
8. **First-party storefront parity.** Does the storefront also need Order before launch, or can it ship later than Beckn-side support?

## Implications

* If first release includes Beckn, then Order moves out of "later phase" status to "earliest phase that makes Beckn work."
* The Bridge sections about correlation, idempotency, and version negotiation assume real flows exist to correlate. Without orders, those facilities are partly hypothetical.
* Payments and fulfillment are the two biggest unknowns; deferring them is reasonable, but only if the architecture admits them later cleanly.

## Dependencies

* Depends on: 05 (BPP identity), 06 (publication), 08 (inventory), 09 (pricing).
* Influences: the entire roadmap. Possibly the most urgent gap to resolve.
