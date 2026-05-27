# Gap 05 — BPP network identity

> **RESOLVED on 2026-05-28** by [ADR-0001 — BPP network identity](../../decisions/0001-bpp-network-identity.md). The active rule lives in `CLAUDE.md` §2.1 and §4.1. This file is preserved for historical context.

## Statement

On the Beckn network, every BPP has a `bpp-id` and `bpp-uri` registered with a network registry, and signs every message it sends. We are described as an "aggregator of small retailers." It is not stated whether we appear on the network as **one BPP listing many providers**, or as **many BPPs (one per store)**. This is the single most consequential Beckn-side decision and shapes everything from message signing to registry interactions to per-store discoverability.

## Current charter coverage

* §2.1 — "exposes those stores to the Beckn network as a single BPP endpoint" suggests one BPP. But this phrasing has not been deliberately ratified.
* §4.1 — Bridge owns "registry / discovery lookups" but no detail on what gets registered.
* No discussion of message-signing identity or per-store cryptographic identity.

## Open questions

1. **Cardinality.** Confirm explicitly: one BPP for the whole platform, or one BPP per store? Hybrid (one BPP per region/category)?
2. **Provider mapping.** If one BPP: each store maps to a `provider` inside Beckn messages. Is the provider ID derived from the store ID? Stable across the lifetime of the store?
3. **Signing keys.** One platform-level signing key, or per-store keys? Implications for key rotation and per-store revocation.
4. **Registry registration.** Who registers the BPP — automated on platform deploy, manual? Per-environment registry endpoints?
5. **Callback URLs.** Single platform-wide callback endpoint, or per-store endpoints? How does the Bridge route inbound callbacks back to the right store context?
6. **BAP targeting.** How does a buyer-side BAP target a specific store after `on_search`? Likely via provider ID — confirm and document.
7. **Reputation / network-level attributes.** Do attributes like ratings, certifications, or network compliance apply to the platform, to the store, or both?

## Implications

* Shapes every outbound Beckn message the Bridge constructs.
* Determines whether the Bridge needs a per-store key-management adapter.
* Affects how store deletion / suspension propagates to the registry.
* If "one BPP" wins, the platform name and reputation become a shared resource across all stores — relevant to onboarding policy.

## Dependencies

* Foundation for: 06 (publication), all of §4.
* Independent of identity/auth gaps.
