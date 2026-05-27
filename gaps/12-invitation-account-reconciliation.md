# Gap 12 — Invitation–account reconciliation

## Statement

§6.4 says the system "must reconcile the email-only path with the User-bound path — if a User signs up using the invited email, they should be linkable to the pending invitation." The word "linkable" carries all the load. The exact rules for linking, identity collision, and edge cases are undefined.

## Current charter coverage

* §6.3 — Invitations target either an existing User or an email address.
* §6.4 — Acceptance "must reconcile" the two paths; must be idempotent.
* No rule for what counts as a match, or what to do when matches are ambiguous.

## Open questions

1. **Match criterion.** Is reconciliation based on a verified email match? Unverified email? Identity-provider subject?
2. **Verification requirement.** Does the system require the invited email to be *verified* before the Membership is created, or is the invitation token itself sufficient proof?
3. **Pre-existing user, same email.** If a User already exists with that email but registered via Google, and the invite is email-only, are they auto-linked on next sign-in? Always, or only on explicit "accept invitation" action?
4. **Multiple pending invitations.** If two stores invite the same email, and the user signs up once, are both invitations linked? Either, both, neither?
5. **Different email at sign-up.** User accepts a Google sign-in whose verified email is different from the invited email. Does the invitation expire / require a new one, or can the user manually claim it?
6. **Sign-up timing.** Is the link established at sign-up time, at first sign-in, or at explicit "accept invitation" click? Different choices have different security trade-offs.
7. **Idempotency.** §6.4 requires acceptance to be idempotent. What's the de-duplication key — invitation token, (user, store) pair, or both?

## Implications

* Resolving this depends on the User and authentication model (Gap 01). Many of the answers vary by whether email verification is mandatory.
* Mistakes here are security-sensitive: linking the wrong identity to a Membership grants store access incorrectly.

## Dependencies

* Depends on: 01 (Identity & Access scope).
* Self-contained within Tenancy otherwise.
