# Gap 15 — First-party idempotency

> **RESOLVED on 2026-05-28** by [ADR-0013 — First-party idempotency](../../decisions/0013-first-party-idempotency.md). Active rules in `CLAUDE.md` §5.18. This file is preserved for historical context.

## Statement

§4.3 establishes that the Beckn Bridge handles protocol-level idempotency for inbound Beckn messages. The charter is silent on idempotency for first-party interfaces (admin UI, storefront, internal APIs). Duplicate submissions from a flaky network, double-clicks, or retry-on-error are everyday occurrences — the Application Layer should have a defined stance.

## Current charter coverage

* §4.3 — Bridge handles "transactional-level idempotency by their protocol-level identifiers."
* §6.4 — Invitation acceptance must be idempotent — but this is a one-off rule, not a general policy.

## Open questions

1. **Default expectation.** Are all Application Layer use cases idempotent by design, or only those explicitly marked as such?
2. **Idempotency key source.** Client-supplied (caller passes a key), server-derived (e.g., hash of inputs), or both?
3. **Key scope.** Per-user, per-store, global? How long is a key valid?
4. **Retention.** How long do idempotency records persist? Where are they stored — in the Application Layer, the Infrastructure Layer, or each context independently?
5. **Conflict semantics.** Same key, different payload — return previous result, error, or overwrite?
6. **Interaction with events.** If a use case is retried after a partial failure (state written, event not emitted), how is the missing event recovered?
7. **Mutating vs. read.** Are reads exempt, or do we want some read-side dedup (e.g., for expensive reports)?

## Implications

* Without an idempotency stance, callers (front-end, internal services) end up implementing ad hoc retry guards or producing duplicate side effects.
* Touches event emission (Gap 13) and consistency (Gap 14) directly.
* Affects the shape of the Application Layer use-case interface.

## Dependencies

* Depends on: 13 (events), 14 (consistency).
