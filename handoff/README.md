# BPP Design — Handoff Package

This directory is the **consolidated design document** for the implementing team. It integrates:

- The **architectural charter** ([`/CLAUDE.md`](../CLAUDE.md)) — principles and rules.
- The **21 Architecture Decision Records** ([`/decisions/`](../decisions/)) — judgment calls with reasoning.
- The **seven sub-design documents** ([`/design/`](../design/)) — per-context detailed models.

into a single navigable narrative organized for reading by an engineer building the system.

## Who this is for

You're an engineer joining the implementing team. You haven't lived through the design process. You want to know:

- **What the system is** and what it does.
- **The architectural rules** you must respect.
- **The shape of each bounded context** you might work on.
- **The cross-cutting patterns** that apply everywhere.
- **Where to look for "why?"** when a decision seems odd.

This document answers those, in roughly that order.

## How to read it

**On first read (1–2 hours):**

1. [§1 Overview](01-overview.md) — what the system is, glossary, big picture.
2. [§2 Architectural principles](02-principles.md) — the rules and the why.
3. [§3 Beckn integration](03-beckn-integration.md) — how the system reaches the network.
4. Skim [§4 Bounded contexts](04-bounded-contexts/README.md) — at least the introductions to each.
5. Skim [§5 Cross-cutting concerns](05-cross-cutting.md) — what applies everywhere.
6. [§7 Open issues](07-open-issues.md) — what was deferred and why.

**For implementing a feature**: jump to the relevant context section in §4 and the relevant cross-cutting topic in §5. Cross-references take you to the ADR or design doc that holds the full reasoning.

## What's authoritative

This document is the **reading view** — it presents the integrated design in a digestible order. For deep "why?" questions, the **ADRs in [`/decisions/`](../decisions/) are the historical record**. ADRs are immutable; this document is a curated synthesis.

When this document and an ADR seem to disagree:
- The **ADR wins** on decision content (what was decided, what was rejected, why).
- This document **wins on current vocabulary and integration** (since ADRs are immutable but the wire vocabulary or terminology has evolved).

## Table of contents

1. [Overview, goals, glossary](01-overview.md)
2. [Architectural principles](02-principles.md)
3. [Beckn integration — the Bridge](03-beckn-integration.md)
4. [Bounded contexts in detail](04-bounded-contexts/README.md)
   - 4.1 [Identity & Access](04-bounded-contexts/4.1-identity.md)
   - 4.2 [Tenancy](04-bounded-contexts/4.2-tenancy.md)
   - 4.3 [Catalog](04-bounded-contexts/4.3-catalog.md)
   - 4.4 [Inventory](04-bounded-contexts/4.4-inventory.md)
   - 4.5 [Promotion](04-bounded-contexts/4.5-promotion.md)
   - 4.6 [Order & Fulfillment](04-bounded-contexts/4.6-order.md)
   - 4.7 [Audit](04-bounded-contexts/4.7-audit.md)
5. [Cross-cutting concerns](05-cross-cutting.md)
6. [Operational stance](06-operational.md)
7. [Open issues and known limits](07-open-issues.md)
8. [References](08-references.md)

## Drafting status

This package is authored section-by-section. Not all sections may exist yet.

| Section | Status |
|---|---|
| README (this file) | Drafted |
| §1 Overview | Drafted |
| §2 Architectural principles | Drafted |
| §3 Beckn integration | Drafted |
| §4 Bounded contexts | Drafted (README + 4.1–4.7) |
| §5 Cross-cutting concerns | Drafted |
| §6 Operational stance | Drafted |
| §7 Open issues | Drafted |
| §8 References | Drafted |
