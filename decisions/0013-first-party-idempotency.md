# ADR-0013: First-party idempotency

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/15-first-party-idempotency.md](../gaps/resolved/15-first-party-idempotency.md)
- **Builds on**: [ADR-0011](0011-domain-events.md) (transactional outbox), [ADR-0012](0012-cross-context-consistency.md) (outbox + inbox)

## Context

[ADR-0011](0011-domain-events.md) and [ADR-0012](0012-cross-context-consistency.md) addressed idempotency on the **subscriber** side — events deliver at least once; subscribers dedup via the inbox table. [ADR-0001](0001-bpp-network-identity.md) (§4.3) addresses **protocol-level** idempotency in the Beckn Bridge for inbound Beckn messages.

What remained: **first-party use case idempotency** — how the system handles duplicate submissions of the same operation from UIs, mobile apps, or internal API clients. Without an explicit pattern, double-clicks, network retries, and mobile reconnections create duplicate state (two Stores with identical data, two Vouchers with the same code, etc.).

Every mutating use case that creates new state needs protection. This ADR establishes the standard pattern.

## Options considered

### Scope of the pattern

**A. Opt-in per use case** (only "blessed" ones support keys). Rejected — relies on memory to mark each new use case; the default unsafe.

**B. Default for all mutating use cases that create new state** — chosen.

### Key source

**A. Client-supplied UUID** — chosen. Stripe-style; deterministic; aligned with user-initiative intent.

**B. Server-derived hash of inputs.** Rejected — timestamps, generated IDs, and other non-deterministic fields make hashes unstable.

### Conflict on payload mismatch

**A. Typed error** — chosen. Surfaces client bugs.

**B. Cached-response-wins** (ignore payload). Rejected — silently masks bugs.

**C. Overwrite.** Rejected — defeats the purpose.

### Storage location

**A. Platform-wide single table.** Rejected — couples contexts.

**B. Per-context table** — chosen. Matches the service-extraction path declared in §2.9.

## Decision

### 1. Default support for mutating use cases

Every Application-Layer use case that **creates new state** (CreateStore, CreateProduct, AcceptInvitation, RedeemVoucher, …) accepts an **optional `idempotency_key`** supplied by the caller.

Use cases that are **naturally idempotent** (those that set state to a target value — `SetStoreStatus(Paused)`, `UpdateProductMedia(...)`) do not need explicit key handling. Calling them twice already produces the same outcome.

If a caller does not supply a key, the use case runs without dedup. Clients are **encouraged** but not required to supply keys for user-initiated mutations.

### 2. Keys are client-supplied UUIDs

The caller (UI / mobile / internal API client) generates a fresh UUID per **logical operation** and sends it with the request. Retries of the same logical operation reuse the same UUID; a different operation uses a different UUID.

The server does not derive or generate keys from request inputs.

### 3. Conflict semantics: typed error on payload mismatch

If the same `(user_id, idempotency_key)` arrives a second time with a **different payload**, the use case returns a typed error: `idempotency_key_reused_with_different_payload`. This surfaces client bugs immediately.

If the same key arrives with the **same payload**, the use case returns the **cached result** of the first execution — no second state change, no second event, no second outbox row.

### 4. Storage

Each context with mutating use cases maintains an `idempotency_records` table:

| Field | Description |
|---|---|
| `user_id` | Acting user; part of the primary key |
| `idempotency_key` | UUID supplied by client; part of the primary key |
| `request_fingerprint` | Stable hash of the canonical request payload (used for the mismatch check) |
| `result_snapshot` | Serialized result of the use case (returned on subsequent calls) |
| `created_at` | When the original execution committed |

Primary key: `(user_id, idempotency_key)`. User-scoped, so different users cannot collide on the same UUID.

Retention: configurable, **default 24 hours**. After expiry, the key can be reused.

### 5. Transactional coherence

The idempotency record is written **in the same DB transaction** as the state mutation(s) and the outbox row(s):

```
BEGIN;
  -- state mutation
  -- outbox row(s)
  -- idempotency_records (key, fingerprint, result_snapshot)
COMMIT;
```

Failure modes:

- **Crash before COMMIT** — no record, no state, no event. Retry runs cleanly as a first execution.
- **Crash after COMMIT but before responding** — retry finds the record, returns the cached result. State change and event already happened; the event will dispatch via the outbox normally.

This makes idempotency keys **transactionally coherent** with state and outbox — the same family of pattern as outbox + inbox.

### 6. Reads are exempt

Read use cases have no side effects. Idempotency keys do not apply to reads.

### 7. Transport carriage

The HTTP layer (and equivalent for non-HTTP transports) carries the key via a standard `Idempotency-Key` header. Internal Application-Layer ports may accept it as a method parameter.

## Consequences

What this commits to:

- Every mutating Application-Layer use case that creates new state accepts an optional `idempotency_key` parameter.
- Each context with such use cases has an `idempotency_records` table written transactionally with state and outbox.
- Conflict on payload mismatch is a typed error — clients must regenerate keys for genuinely new operations.
- Idempotency records are user-scoped; cross-user collisions are impossible.
- Keys are carried via the `Idempotency-Key` HTTP header at the transport boundary.
- The system has full at-least-once safety across three planes: producers (outbox), subscribers (inbox), and clients (idempotency keys).

What this defers:

- **Exact retention window** — operational; default 24 hours.
- **Cleanup mechanism** for expired records — operational (background job or DB TTL).
- **Canonical fingerprint algorithm** for payload comparison — implementation detail; should normalize (sort keys, trim whitespace, stable JSON ordering).
- **Client SDK helpers** for key generation — UX / SDK concern.
- **Idempotency for non-creating mutations** (updates, deletes) — naturally idempotent by setting-to-value semantics; no explicit key needed.

What this makes harder:

- **Clients that don't supply keys** still risk duplicates from retries. The pattern is opt-in from the client side; the server cannot enforce supply.
- **Reusing one key across logically distinct operations** is a programming error and surfaces as the typed error in §3.
- **Read-after-write retries** — a client retrying with the same key gets the cached result, not a fresh read. For read-after-write consistency, clients must issue a separate read.

## References

- [gaps/resolved/15-first-party-idempotency.md](../gaps/resolved/15-first-party-idempotency.md)
- [ADR-0011](0011-domain-events.md) (outbox), [ADR-0012](0012-cross-context-consistency.md) (outbox + inbox)
- [ADR-0001](0001-bpp-network-identity.md) §4.3 (Bridge protocol-level idempotency — separate concern)
- CLAUDE.md §5.18 (this ADR's rules)
