# Gap 02 — Platform-scoped roles

> **RESOLVED on 2026-05-28** by [ADR-0002 — Authorization tiers and the capability matrix](../../decisions/0002-authorization-tiers-and-matrix.md). The active rule lives in `CLAUDE.md` §5.3–§5.7. This file is preserved for historical context.

## Statement

§5.4 declares that platform-scoped roles exist and must be modeled separately from store-scoped roles, but does not say what those roles are, who holds them, what they can do, or how they are assigned. Until the platform admin model is defined, any operation that crosses tenants (suspending a store, reviewing publication, running audits) has no actor.

## Current charter coverage

* §5.4 — "Platform-scoped roles (e.g., platform operators, support staff) — apply across the system." Distinguished from store-scoped roles, no further detail.
* §5.2 — "There is no 'global' listing of tenant-owned data **outside of platform-level admin operations**" — implies such operations exist but never names them.

## Open questions

1. **Catalog of platform functions.** What can platform actors actually do? Candidates: suspend/restore a store, view all users, review store applications, approve Beckn publication, run audits, manage protocol versions, manage platform settings.
2. **Role granularity.** Is there a single `PlatformAdmin`, or are there distinct roles (Operator, Support, Compliance, Network)?
3. **Assignment mechanism.** Who grants platform roles? Is the first platform admin bootstrapped via configuration / fixture, or is the granting flow fully in-band?
4. **Modeling stance.** Is a platform role just another row in the same authorization tables (with a "scope = platform" marker), or a wholly separate entity (e.g., `PlatformMember`)? §5.4 says "separate concepts" but doesn't say whether that means separate fields or separate tables.
5. **Cross-tenant action semantics.** When a platform admin acts on a store (e.g., suspends it), are they recorded as the actor in the store's audit trail? Do they need to be a Member of that store, or does platform role grant access without Membership?
6. **Impersonation.** Can a platform admin "act as" a store admin for support purposes? If yes, is the action attributed to the admin, the user, or both?

## Implications

* Authorization (Gap 03) must compose platform-scope and store-scope decisions cleanly.
* Store lifecycle (Gap 04) operations like suspension and forced deletion need platform actors.
* Beckn publication review (Gap 06) likely belongs to a platform role.

## Dependencies

* Blocks: 03 (authorization composition), 04 (store lifecycle actors), 06 (publication approval).
* Depends on: 01 (User model).
