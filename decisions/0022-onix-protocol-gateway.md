# ADR-0022: ONIX as Beckn protocol gateway

- **Status**: Accepted
- **Date**: 2026-05-31
- **Partially supersedes**: [ADR-0001](0001-bpp-network-identity.md) (signing key custody, registry interactions, Beckn callback endpoint exposure); [ADR-0017](0017-order-and-fulfillment.md) (direct CDS publishing from the Bridge)

## Context

The BPP's interaction with the Beckn / ION network was originally designed (in [ADR-0001](0001-bpp-network-identity.md) and [ADR-0017](0017-order-and-fulfillment.md)) as a direct integration: the Bridge signs outbound messages, verifies inbound signatures, validates payloads against `ion-specs` schemas, looks up participants in the Beckn registry, and publishes catalog updates to CDS — all from BPP-owned code, with the signing key and registry credentials held in the BPP's secrets manager.

The network is now intermediated by **ONIX** — a Beckn Protocol Server (BPP-PS) that sits between the BPP and the network. ONIX is the network-visible BPP from the registry and BAP perspective: signed messages on the wire come from ONIX, registry interactions go through ONIX, and CDS publishes are routed via ONIX. The BPP behind ONIX speaks Beckn JSON, but offloads the network-level cryptographic and protocol machinery to ONIX.

This architectural shift simplifies the BPP's Bridge — but not as much as a naive reading suggests. Several protocol concerns stay with the BPP, including inbound signature re-verification (because the CounterSignature in the BPP's Ack attests the BPP itself authenticated), ION error code mapping (the BPP both consumes ONIX-returned codes and produces them in Nack responses), and Ack/AckNoCallback/Nack construction. The net effect is a re-shaped Bridge, not a vestigial one.

## Options considered

### Option A — Stay direct: BPP integrates with the network without a gateway

- **Summary.** Keep the design from ADR-0001 + ADR-0017: BPP signs, verifies, validates against `ion-specs`, holds the signing key + registry credentials, and publishes to CDS directly.
- **Pros.** Simpler topology (one fewer component). Full control over crypto + validation. No vendor dependency for the network path.
- **Cons.** BPP code carries protocol-level concerns (signing, registry coordination, version negotiation) that aren't business-meaningful. PCI/security surface area is larger. Key rotation is an in-band BPP concern. Operationally heavier.
- **Implications.** Bridge folder includes signing, verification, registry-client, CDS-client modules. BPP secrets include signing key + registry credentials.

### Option B — Use ONIX as a transparent protocol gateway — chosen

- **Summary.** A vendor-provided ONIX binary, deployed per-BPP, handles signing of outbound messages, verification of inbound signatures, schema validation, registry interactions, and CDS publish on the BPP's behalf. The BPP and ONIX communicate over plain HTTPS POST with the full Beckn JSON envelope; ONIX is the network-visible endpoint.
- **Pros.** Signing key and registry credentials move out of the BPP's secrets. Crypto + ion-specs schema validation move out of BPP code. Vendor handles network-level protocol concerns. The BPP's surface area shrinks to domain ↔ wire mapping + Beckn-state-dependent business logic.
- **Cons.** New operational dependency (ONIX availability). Vendor binary introduces a black box for parts of the protocol surface. The "transparent signer + router" abstraction is leaky in places that matter (notably: BPP must still re-verify inbound signatures to honor the CounterSignature attestation).
- **Implications.** Bridge folder loses signing.ts, verification.ts (for outbound), registry-client, direct CDS client; gains an ONIX client and an ION error registry consumer. The wire-state mapping, mapping registry, inbound parsing, Ack/Nack construction, and inbound signature re-verification stay.

### Option C — Build an in-house gateway equivalent to ONIX

- **Summary.** Build a BPP-owned protocol server that does what ONIX does, but in-house.
- **Pros.** Full control over the protocol path. No vendor dependency.
- **Cons.** Re-implements ONIX. Significant engineering investment for capability that's available off-the-shelf. Maintenance burden for crypto + registry + CDS code.
- **Implications.** Rejected — does not justify the cost relative to Option B.

## Decision

Adopt Option B: **ONIX as a transparent protocol gateway, deployed per-BPP**.

### 1. Responsibility split

| Concern | BPP (Bridge) | ONIX |
|---|---|---|
| Domain ↔ wire mapping (Order ↔ Contract, etc.) | ✓ | — |
| Outbound message construction (Beckn v2 envelope, `context.bpp_id`, `context.bpp_uri`, payload) | ✓ | — |
| Outbound message signing (`Authorization` header on the wire) | — | ✓ |
| Inbound signature verification (initial, at network door) | — | ✓ |
| **Inbound signature re-verification** (for CounterSignature attestation) | ✓ | — |
| ion-specs schema validation at network door | — | ✓ |
| Beckn registry interactions | — | ✓ |
| CDS publishing | — | ✓ (forwarded from BPP) |
| TTL enforcement at network door (ION-1003) | — | ✓ |
| Duplicate-message dedup at network door (ION-1004) | — | ✓ |
| Inbound `bridge_inbox` dedup by `(transaction_id, message_id)` | ✓ | — |
| Correlation across async callback chain | ✓ | — |
| Locale resolution of `LocalizedText` per `context.language` | ✓ | — |
| `bridge_outbox` retry on transient (5xx) ONIX failures | ✓ | — |
| Ack / AckNoCallback / Nack construction with CounterSignature | ✓ (see §6) | — |
| ION error code mapping (domain ↔ ION-XXXX) | ✓ | — |
| Signing key custody | — | ✓ (vendor secrets) |
| Registry credentials custody | — | ✓ (vendor secrets) |

### 2. Wire protocol between BPP and ONIX

- **Full Beckn JSON envelope.** No internal wrapper. The BPP constructs and consumes the same envelope shape as the network sees.
- **Plain HTTPS POST.** No API key, no mTLS, no bearer token, no HMAC between BPP and ONIX.
- **BPP-Identifier fields filled by the BPP.** `context.bpp_id` and `context.bpp_uri` are set by the BPP. `bpp_uri` IS the ONIX network-facing endpoint — the value the BPP sets in its outbound envelopes matches the URL the network sees.
- **Protocol version targeted directly.** BPP constructs Beckn v2.0.0 envelopes. ONIX is a transparent signer + router; it does not version-translate.

### 3. Outbound flow (BPP → ONIX → network)

```
BPP                                       ONIX                              Network
 │                                          │                                  │
 │  POST <onix-url>/<action>                │                                  │
 │  (full Beckn envelope; no Authorization) │                                  │
 ├─────────────────────────────────────────►│                                  │
 │                                          │  Sign with platform key;         │
 │                                          │  add Authorization header;       │
 │                                          │  forward to BAP callback URL     │
 │                                          ├─────────────────────────────────►│
 │                                          │                                  │
 │  ◄── 2xx (accepted) OR 4xx/5xx (failure) │                                  │
 │                                          │                                  │
```

- The BPP omits the `Authorization` header on outbound POSTs. ONIX adds it.
- On 5xx (transient — including ION-9001), the BPP retries from `bridge_outbox` with exponential backoff.
- On 4xx (structural / policy / signing-side errors), the BPP marks the row `dead` in `bridge_outbox` and emits a domain event for ops review.

### 4. Inbound flow (network → ONIX → BPP)

```
Network                  ONIX                                BPP
 │                         │                                   │
 │  Signed Beckn message   │                                   │
 ├────────────────────────►│                                   │
 │                         │  Verify signature against         │
 │                         │  registry; validate schema;       │
 │                         │  check TTL + duplicates;          │
 │                         │  forward if all pass              │
 │                         ├──────────────────────────────────►│
 │                         │  (full envelope including         │
 │                         │   BAP's Authorization header,     │
 │                         │   for re-verification)            │
 │                         │                                   │
 │                         │                                   │  Re-verify signature
 │                         │                                   │  (for CounterSignature
 │                         │                                   │   attestation);
 │                         │                                   │  dedup by (tx_id, msg_id);
 │                         │                                   │  parse + map → domain;
 │                         │                                   │  invoke Application Layer;
 │                         │                                   │  construct Ack/Nack with
 │                         │                                   │  CounterSignature
 │                         │                                   │
 │  ◄────────── Ack/Nack (with CounterSignature) ──────────────│
 │                         │                                   │
```

- The BPP receives the full original Beckn envelope including the BAP's `Authorization` header.
- **Inbound signature re-verification is mandatory.** The BPP's CounterSignature in the Ack attests that *the BPP itself* authenticated the request; relying on ONIX's verification alone breaks this guarantee.
- The BPP responds synchronously with one of `Ack`, `AckNoCallback`, or `Nack` — every inbound gets a typed response (silent drop is forbidden by the protocol).

### 5. Error semantics — ION-XXXX registry

The BPP both consumes errors from ONIX and produces errors back through ONIX. Both directions use the ION-XXXX error code registry from `ion-specs/errors/`.

**Registry structure** (from `ion-specs/errors/README.md`):

| Range | Category |
|---|---|
| ION-1xxx | Transport (signing, identity, TTL, duplicate messageId, domain) |
| ION-2xxx | Catalog |
| ION-3xxx | Transaction |
| ION-4xxx | Fulfillment |
| ION-5xxx | Post-order |
| ION-6xxx | Settlement |
| ION-7xxx | Network (policy violations) |
| ION-8xxx | Schema |
| ION-9xxx | System (ION infrastructure) |

Each entry pins an `http_status` and carries `title`, `description`, `affected_field`, `affected_apis`, and `resolution`.

**Wire format**: `{ errorCode: "ION-XXXX", errorMessage: string }`.

**BPP responsibilities**:
- **Load the registry at startup.** Bridge consumes `errors/registry.json` (generated from category YAMLs). Look up codes on the fly; do not hand-code mappings.
- **Map ONIX failure responses to retry decisions**: 5xx (ION-9001, ION-9002 timeout, etc.) → retry from outbox; 4xx → fatal (mark `dead`, emit event).
- **Map domain errors to ION-XXXX codes** for Nack responses. Bridge's `mapping-registry/errors.ts` becomes a domain-error → ION-XXXX projection.
- **Defense-in-depth re-checks**: even though ONIX rejects most structural failures, the BPP validates inbound against `ion-specs` once parsed — to catch ONIX bugs or registry drift.

### 6. Inbound signature re-verification

The BPP must re-verify the BAP's signature on inbound messages even though ONIX has already verified. The reason:

> The CounterSignature in the BPP's `Ack` attests that the **BPP itself** authenticated the request. Without a re-verification step, the attestation is implicit ("ONIX told us this is OK") rather than explicit ("we checked the signature ourselves").

Implementation:

- The Bridge keeps a slim verification module (`bridge/protocol/verification.ts`).
- The module verifies the `Authorization` header on the inbound request against the BAP's public key.
- **N2 (open)**: how the BPP obtains the BAP's public key — either via its own registry client, or because ONIX forwards the key as a header / metadata on the request. Resolution deferred (see §10).

### 7. Ack / AckNoCallback / Nack with CounterSignature

Every inbound message receives a synchronous typed response with a CounterSignature attesting the BPP authenticated.

- **Ack** — the request is accepted and a callback (e.g., `/on_select`) will follow asynchronously.
- **AckNoCallback** — the request is accepted but no callback will follow (used for status queries that complete synchronously).
- **Nack** — the request is rejected at the BPP level (state-dependent failure that ONIX couldn't catch — e.g., ION-3xxx transactional, some ION-7xxx policy).

The CounterSignature accompanies all three.

- **N1 (open)**: whether the BPP signs the CounterSignature itself (requiring a narrow BPP-held key) or whether ONIX adds the CounterSignature on the synchronous response path (in which case the BPP truly has no signing key). Resolution deferred (see §10).

### 8. Operational shape

- **ONIX is a vendor-provided binary**, deployed per-BPP. Each BPP has its own ONIX instance.
- **`bpp_uri` is the ONIX network-facing endpoint.** ONIX can be deployed anywhere (sidecar, sibling, VPC-internal, dedicated host) — the BPP just needs to be able to reach the BPP-facing endpoint of ONIX (which is private, since there's no auth), and the network needs to reach the network-facing endpoint (which is public, signed-traffic-only).
- **No auth between BPP and ONIX.** Network-level isolation is the security model. The BPP-facing endpoint of ONIX MUST NOT be on the public internet.
- **Configuration per environment**: ONIX endpoint URL (BPP-facing) is the only BPP-side env var for ONIX integration. Signing key and registry credentials are ONIX's config, not the BPP's.
- **Zero-downtime ONIX restarts.** During an ONIX restart, BPP outbound POSTs get transient 5xx errors; the `bridge_outbox` retries with backoff (matching ION-9001 retry semantics). No special "drain BPP" pattern needed.
- **Operational ownership**: as a vendor binary, ONIX is operated alongside the BPP. Upgrades, observability, and config are handled per the vendor's guidance, not in our developer-guide.

### 9. What survives unchanged from ADR-0001 + ADR-0017

- **Network identity model.** The platform appears on the Beckn network as a single BPP. Stores project as Beckn `provider` nodes inside BPP responses. Per-store cryptographic isolation is not provided.
- **Provider-ID derivation.** `provider` IDs on the wire are derived deterministically from internal store identifiers.
- **CDS-mediated discovery model.** The BPP does NOT field `/discover` directly; discovery is `BAP → CDS → BPP transactional`. Now: `BAP → CDS (via ONIX) → BPP via ONIX`.
- **Catalog publishing flow.** The BPP triggers catalog publishes on relevant events; only the destination changes (was: BPP → CDS direct; now: BPP → ONIX → CDS).
- **`bpp_id` is BPP-side config.** Same env var; ONIX is transparent for this field.
- **Order state machine, Quote semantics, fulfillment progression** ([ADR-0017](0017-order-and-fulfillment.md) §1–§3, §5–§7) — all unchanged.
- **`bridge_inbox` + `bridge_outbox` tables** as designed.

## Consequences

### Source-level impact

**Bridge folder changes** (per [§2.5 of `02-repo-layout.md`](../developer-guide/02-repo-layout.md)):

| Removed | Reason |
|---|---|
| `bridge/protocol/signing.ts` | ONIX signs outbound |
| (no change) `bridge/protocol/verification.ts` | Stays — needed for inbound re-verification |
| `platform-infra/beckn-network/cds-client/` | ONIX handles CDS publish |

| Modified | Change |
|---|---|
| `platform-infra/beckn-network/` | Becomes a single thin `onix-client/` adapter that POSTs to ONIX |
| `bridge/protocol/verification.ts` | Simplified — only inbound re-verification, no outbound signing concerns |
| `bridge/catalog-publisher/` | Target changes from CDS endpoint to ONIX endpoint (same JSON payload) |
| `bridge/mapping-registry/errors.ts` | Now backed by `ion-specs/errors/registry.json` — domain error → ION-XXXX mapping consumes the registry |

| New | What it adds |
|---|---|
| `bridge/protocol/ion-error-registry.ts` | Loads `errors/registry.json` at startup; exposes lookup; maps ONIX failure responses to retry/fatal decisions |
| `bridge/protocol/ack-builder.ts` | Builds Ack / AckNoCallback / Nack responses with CounterSignature (subject to N1 resolution) |

### Configuration impact

**Removed from BPP secrets / config** (now ONIX's responsibility):
- Beckn signing key
- Beckn registry credentials
- CDS endpoint URL + credentials

**Stays in BPP config**:
- `BPP_ID` (filled into `context.bpp_id`)
- `BPP_URI` (filled into `context.bpp_uri` — value equals ONIX network-facing URL)

**New BPP config**:
- `ONIX_ENDPOINT` (the BPP-facing URL of ONIX)

### Documentation cascade (handled in subsequent PRs)

- **Partial-supersession markers** added to [ADR-0001](0001-bpp-network-identity.md) and [ADR-0017](0017-order-and-fulfillment.md), per [CLAUDE.md §10.2](../CLAUDE.md).
- **`handoff/03-beckn-integration.md`** rewritten to reflect the four-actor (BAP ↔ ONIX ↔ BPP, ONIX ↔ CDS) topology. Signing / verification sections become "verified by ONIX, re-verified by BPP for CounterSignature attestation." CDS-publish section becomes "BPP publishes to ONIX; ONIX forwards to CDS."
- **`developer-guide/06-context-playbooks/6.7-beckn-bridge.md`** updated for the new Bridge folder structure (onix-client, ion-error-registry, ack-builder).
- **`developer-guide/08-deployment.md`** updated to drop signing key + registry credentials from the BPP secrets list; add ONIX as vendor-binary deploy concern; clarify that `bpp_uri` is the ONIX network-facing URL.
- **`developer-guide/09-runbooks.md`** updated: "rotate Beckn signing key" runbook becomes "vendor operation, coordinate with ONIX-side"; new runbook for "ONIX unreachable triage"; new runbook for "ION-XXXX error registry sync from ion-specs."

### Risks

- **ONIX availability is a SPOF.** If ONIX is down, the BPP is off the network even if otherwise healthy. Mitigated by per-BPP deployment (no shared-tenant outage), zero-downtime restarts, and outbox retry.
- **Vendor binary opacity.** ONIX's behavior on edge cases (e.g., partial registry sync, ION-XXXX gap codes) is determined by the vendor. We mitigate via defense-in-depth re-validation in the BPP for the cases we care about.
- **CounterSignature semantics (N1 open).** Until resolved, the Bridge architecture assumes inbound re-verification stays in BPP code. If N1 resolves to "ONIX adds CounterSignature," the Bridge simplifies further.

## Open follow-ups

These do not block this ADR's acceptance; they are operational specifics to resolve before / during Phase 6 (Bridge) implementation per [`05-phases.md`](../developer-guide/05-phases.md).

### N1. CounterSignature signing locus

Two interpretations of "ONIX is transparent signer + router":

- **(a)** ONIX signs only async outbound messages. The BPP holds a narrow signing key for CounterSignatures on synchronous Ack/Nack responses.
- **(b)** ONIX signs both async outbound and synchronous response CounterSignatures. The BPP has no signing key at all.

Resolution affects whether the BPP's secrets manager holds a signing key. Default working assumption: **(b)** (more consistent with "transparent signer"); to be confirmed before Phase 6.

### N2. BAP public key for inbound re-verification

Two implementation paths:

- **(a)** The BPP runs its own ION registry client to look up BAP public keys on demand.
- **(b)** ONIX includes the BAP's public key (or registration context) as a header / metadata on the forwarded request.

Default working assumption: **(b)** (consistent with ONIX-as-gateway pattern; avoids duplicating registry client logic in BPP). To be confirmed before Phase 6.

### N3. ION-XXXX gap codes

The user's review noted gaps in the existing ION-XXXX registry worth proposing additions for:
- Signing-key unavailable (distinct from generic ION-9001 internal error).
- Registry-timeout-vs-miss split (currently both fall under ION-9001 / ION-9002).
- CDS unreachable specifically.
- Forward-to-downstream-failed (when ONIX cannot reach the BAP for outbound).

These should be submitted via the PR flow in `ion-specs/errors/README.md`. Tracked as a Phase 6 prerequisite, not blocking this ADR.

## References

- [ADR-0001](0001-bpp-network-identity.md) — partially superseded by this ADR (signing key, registry, callback endpoint sections).
- [ADR-0017](0017-order-and-fulfillment.md) — partially superseded by this ADR (direct CDS publishing).
- `ion-specs/errors/README.md` — ION-XXXX error code registry structure and contribution process.
- `ion-specs/errors/registry.json` — generated registry consumed by the Bridge at startup.
- [§4 of `CLAUDE.md`](../CLAUDE.md) — Beckn integration strategy (Bridge responsibilities to be reframed post-ADR).
