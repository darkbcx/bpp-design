# Design: <Topic>

- **Status**: Draft
- **Last updated**: YYYY-MM-DD
- **Backed by ADRs**: <!-- ADR-NNNN, ADR-NNNN -->

## Purpose

What this document covers. What it explicitly does **not** cover (and where to look instead).

A design doc exists when a topic is too detailed to live inside CLAUDE.md but too connected to live only inside ADRs. Examples: the full Catalog model, the Order lifecycle, the Identity context's internal structure.

## Position within the architecture

Which bounded context(s) does this design live in? Which layers does it span? Reference the relevant CLAUDE.md sections rather than re-stating their content.

## Concepts

The domain concepts introduced or shaped by this design. **Definitions, not schemas.** Use the charter's vocabulary. Capture each concept's reason to exist before its attributes.

## Relationships

How the concepts relate to each other and to other contexts. Cross-context references must be by identifier, never by intimacy with another context's internals (§2.5).

## Lifecycles & invariants

State machines, transition rules, things that must always hold. If a concept has more than two states, draw the transitions explicitly.

## Boundary contracts

What this subsystem exposes to other parts of the system:

- **Use cases** the Application Layer offers callers.
- **Domain events** this design emits, with semantics.
- **Queries / read models** consumers may rely on.

What this design does **not** expose (internal models, intermediate states, private events).

## Beckn projection notes

If applicable: which Beckn concepts this domain projects to. Mapping discipline lives in §4 and inside the Bridge — this section only identifies *which projections will be needed*, not how they are built.

## Open questions

Anything unresolved within this design. Link to gap files where appropriate.

## References

- ADRs that back this design.
- Charter sections.
- Reference projects (`bitemycart`, `ion-specs`).
