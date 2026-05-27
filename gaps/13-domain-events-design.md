# Gap 13 — Domain events design

## Statement

Domain events appear repeatedly in the charter — as inter-context communication (§2.6), as audit triggers (§5.3, §6.5, §6.6), and as Application-Layer outputs (§2.4). But "domain event" is used as a primitive without being defined. Are events transient or persisted? Do they have a versioned contract? When are they emitted relative to the state change?

## Current charter coverage

* §2.4 — Application Layer "coordinates domain-event publication."
* §2.6 — Domain events are one of two inter-context communication channels.
* §2.6 — Events must describe "what happened in domain terms — never transport, never UI, never Beckn vocabulary."
* §5.3, §6.5, §6.6 — Various "events" are described as auditable.

## Open questions

1. **Persistence.** Are events stored (event log / event store), or transient (in-memory bus, drop on consumer failure)?
2. **Public vs. private.** Is the set of events a context emits a versioned, documented contract (other contexts depend on them) or an internal implementation detail subject to change?
3. **Emission timing.** Emitted within the same transaction as the state change (transactional outbox)? Immediately after commit? Asynchronously batched?
4. **Naming convention.** Domain past-tense (`ProductPublished`, `MembershipAccepted`) — confirm. Namespaced by context?
5. **Schema and evolution.** Are event payloads strongly typed? Versioned? Backwards-compatible by convention?
6. **Replay semantics.** Can subscribers replay historical events on cold start, or do they only see live events?
7. **Relationship to audit.** Is the event log also the audit log (Gap 16), or are they separate?
8. **Delivery guarantees.** At-least-once, exactly-once, at-most-once? Idempotency requirement on subscribers?

## Implications

* The choice between transient and persisted events affects whether contexts can recover from missed events.
* If events are a public contract, evolving them carelessly breaks subscribers.
* Touches both inter-context comms (Gap 14) and audit (Gap 16).

## Dependencies

* Influences: 14 (cross-context consistency), 16 (audit).
* Should be resolved before any context starts emitting events.
