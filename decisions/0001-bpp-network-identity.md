# ADR-0001: BPP network identity

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/05-bpp-network-identity.md](../gaps/resolved/05-bpp-network-identity.md)

## Context

The platform is a multi-tenant aggregator of small retailers exposing them to the Beckn network. A BPP on the network is identified by a `bpp-id` and `bpp-uri`, signs every outbound message with a registered key, and is discoverable through the network registry. The platform must decide whether to present itself as **one BPP** (containing many providers internally) or **many BPPs** (one per store). This decision shapes message signing, callback URLs, registry interactions, provider targeting, and how reputation accrues. It is the single most consequential Beckn-side decision and is a precondition for every detail in §4 of the charter.

## Options considered

### Option A — One BPP, stores as providers

- **Summary.** The platform registers as a single BPP. Each store projects as a Beckn `provider` node inside the BPP's responses. Products and categories live under the provider.
- **Pros.** Simple operational footprint: single network identity, single signing key, single callback endpoint, single registry record. Matches Beckn's `provider` concept exactly. Onboarding a new store is purely internal — no external registry action required. BAPs already select a specific seller by provider ID, so per-store targeting is solved by the protocol.
- **Cons.** Reputation, ratings, and compliance attributes accrue at platform level. A misbehaving store damages the BPP's standing on the network. Per-store cryptographic isolation is not possible.
- **Implications.** Bridge owns the platform key and a single inbound callback. Provider IDs must be derivable from store identifiers. Quality control at onboarding becomes a platform responsibility.

### Option B — One BPP per store

- **Summary.** Each store registers independently with its own BPP identity, signing key, callback URL, and registry entry.
- **Pros.** Per-store reputation isolation. Per-store key rotation and revocation. Stores have full sovereignty on the network.
- **Cons.** O(N) registry entries to maintain. O(N) signing keys to manage. Each store's callback URL is publicly visible on the network, leaking tenancy structure. Each onboarding requires a network-level action. Contradicts the "aggregator of small retailers" framing — small retailers are precisely the segment least equipped to handle independent network identity.
- **Implications.** Bridge becomes a key-management system. Registry interactions become continuous rather than one-time.

### Option C — Hybrid (tiered)

- **Summary.** Most stores aggregated under one BPP; premium or opted-in stores get their own BPP identity.
- **Pros.** Future flexibility for high-volume sellers.
- **Cons.** Dual operational model. Unclear promotion / demotion policy. Doubles the Bridge surface for marginal benefit at this phase. Premature optimization.

## Decision

**Option A. The platform operates as a single BPP. Stores project as Beckn `provider` nodes.**

Specifically:

- The platform appears on the Beckn network as one BPP, identified by a single `bpp-id` and `bpp-uri`.
- Each store projects as a Beckn `provider` inside the BPP's responses. A store's products and categories surface as `items` and `category` blocks under its provider node.
- The Beckn-facing **provider ID** is derived deterministically from the internal store identifier using a stable, opaque scheme. There is no separate mapping table — the same store always projects with the same provider ID.
- **One platform-level signing key** is used for all outbound Beckn messages. Per-store cryptographic identity is not introduced.
- **One platform-wide Beckn callback endpoint** receives all inbound asynchronous responses. The Bridge routes each inbound message to the correct store context using protocol-level identifiers (transaction ID, message ID, provider reference) — never by URL shape.
- **Reputation and network-level attributes** are platform-owned. Surfacing per-store reputation views back to store owners is explicitly deferred.
- **Registry registration is treated as an external operational input.** The `bpp-id`, `bpp-uri`, signing key material, and registry credentials are supplied to the Bridge as configuration. The Bridge does not register or update the BPP entry on the network.

## Consequences

What this commits the system to:

- The Bridge is the **sole** owner of the platform signing key, the inbound callback endpoint, and the provider-ID derivation scheme.
- The Bridge maintains a protocol-level routing capability that maps inbound Beckn context (transaction ID, provider reference) back to the originating store. This routing concern lives entirely in the Bridge, not in any domain context.
- The domain `Store` entity is and remains unaware of its Beckn-facing provider ID. That projection lives only in the Bridge.
- A store's Beckn identity is bound to its store identity for life. "Resetting" Beckn presence requires creating a new store — a deliberate consequence of deterministic derivation.
- Quality control at store onboarding becomes a platform responsibility, because all stores share the BPP's reputation.
- BPP registration is assumed pre-existing. The Bridge consumes registry credentials but does not own their lifecycle.

What this defers (does not decide):

- The internal "Catalog" entity question — a store may internally have one or more catalogs of products. The shape of this entity is owned by [Gap 07 (Catalog modeling scope)](../gaps/07-catalog-modeling-scope.md). This ADR commits only to the *projection*: a store's products surface under its provider node.
- Per-store reputation visibility — deferred to a later phase.
- Operational specifics of registry registration (who, when, per environment) — owned by operations.

What this makes harder:

- Per-store cryptographic isolation. Not achievable under Option A.
- Per-store reputation isolation on the network. Inherently impossible with one BPP.

If either of these becomes required later, this ADR must be superseded by a new one rather than amended.

## References

- [gaps/resolved/05-bpp-network-identity.md](../gaps/resolved/05-bpp-network-identity.md)
- CLAUDE.md §2.1 (system framing), §4.1 (Bridge exclusive responsibilities)
- `ion-specs` — `provider`, `catalog`, `context`, signing, and registry concepts
