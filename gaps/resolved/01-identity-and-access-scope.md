# Gap 01 — Identity & Access scope

> **RESOLVED on 2026-05-28** by [ADR-0009 — Identity & Access (external IdP integration)](../../decisions/0009-identity-and-external-idp.md). Full entity model in [`design/identity.md`](../../design/identity.md). Active rules in `CLAUDE.md` §5.15. This file is preserved for historical context.

## Statement

The `Identity & Access` bounded context is named in §2.5 but its internal model is undefined. Sign-up / sign-in are mentioned in the project overview, and "future support for external identity providers is expected." That is the entirety of the guidance. Before any other context can rely on a `User` identifier, the shape of `User` and its authentication surface must be agreed.

## Current charter coverage

* §2.5 — "Identity & Access — users, credentials, sessions, external identity providers, authentication tokens."
* §3.2 — User is implied to be an entity with stable internal identity.
* §5.3 — A User's identity is global; their capabilities are per-store.
* Project overview — "Users can sign up and sign in. Future support for external identity providers is expected."

## Open questions

1. **User shape.** Is a `User` just an authentication subject (id + credentials), or does it carry profile data (display name, avatar, locale)? If profile, who owns it — Identity & Access or a separate Profile concept?
2. **Authentication methods at launch.** Email + password? Magic link only? Both? Or IdP-only from day one with no native credentials?
3. **Email verification.** Required before sign-in? Required before becoming a member of a store? Required at all?
4. **External IdP linking model.** Can one User have multiple credentials (password + Google + Apple), or is it strictly one identity = one method?
5. **Account merging.** If a User signs up with email/password, then later authenticates with Google using the same email, are these the same User? Auto-merge, prompt-to-link, or treat as distinct?
6. **Session model.** What represents a session — opaque token, JWT, server-side record? Lifetime? Refresh semantics? Revocation (logout-everywhere)?
7. **User deletion.** Is deletion possible? What happens to Memberships, audit trails, and orders owned by a deleted User?

## Implications

* Every other context references `User` by identifier. Without a stable User model, Memberships, Invitations, audit entries, and order ownership cannot be designed.
* The answer to (4) and (5) directly shapes the invitation-reconciliation rules (Gap 12).
* The session model influences how first-party adapters carry identity into the Application Layer.

## Dependencies

* Blocks: 12 (invitation reconciliation), 17 (PII).
* Independent of Beckn integration gaps.
