# ADR-0010: Invitation–account reconciliation

- **Status**: Accepted
- **Date**: 2026-05-28
- **Resolves gap**: [gaps/resolved/12-invitation-account-reconciliation.md](../gaps/resolved/12-invitation-account-reconciliation.md)
- **Builds on**: [ADR-0009](0009-identity-and-external-idp.md) (IdP-verified email is the canonical link), [ADR-0002](0002-authorization-tiers-and-matrix.md) (role model)

## Context

Invitation acceptance is the atomic transition from `Invitation(Pending)` to `Membership(Active)` (CLAUDE.md §6.4). Until ADR-0009 was accepted, the "linkable" semantics — how an invitation finds its target User — were ambiguous because the system supported the abstract notion of "email-only invitations" without a clear identity model.

With ADR-0009 in place, all User identity comes from an external IdP via Just-in-Time provisioning, and the IdP's verified email is the canonical link. The reconciliation logic collapses to a small, decidable rule.

## Options considered

### Match criterion

**A. IdP-verified email match (case-insensitive)** — chosen.
**B. Match by IdP subject_id.** Rejected — the inviter doesn't have visibility into the recipient's IdP subject; only the recipient's email is known.
**C. Fuzzy / heuristic email matching.** Rejected — security risk.

### Linking trigger

**A. Always explicit user action ("Accept" button)** — chosen. Invitations grant store access; informed consent matters.
**B. Auto-link on any sign-in if email matches.** Rejected — surprising; the User may not even know about the invitation.

### Email mismatch handling

**A. Reject with typed error; issuer must re-invite** — chosen.
**B. Manual claim flow with alternate-email proof.** Rejected — premature complexity for a rare case.

## Decision

### 1. Reconciliation rule

An `Invitation` can be Accepted by a User if and only if all of the following hold:

- `User.email == Invitation.email` (case-insensitive comparison on the IdP-canonical email).
- `User.email_verified_at` is set (per ADR-0009, this reflects the IdP's `email_verified` claim).
- `User.status == Active`.
- `Invitation.state == Pending` and `Invitation.expires_at > now`.

If any condition fails, `AcceptInvitation` returns a **typed error** (no state change). No fallback path, no manual claim.

### 2. Discovery paths

A recipient discovers their Invitations in two ways:

- **Email link.** The platform sends the invitee an email containing a link to `/invitations/<token>`. Clicking it shows the invitation. If not signed in, the visitor is redirected to IdP sign-in (carrying a return URL); after sign-in (JIT-provisioned if first time), they land back on the invitation page and choose Accept or Decline.
- **Dashboard panel.** Signed-in Users see all `Pending` invitations whose `email` matches their IdP-verified `email` (case-insensitive) in their account dashboard. Same Accept/Decline actions, same rule.

Both paths converge on the same `AcceptInvitation` use case — same rule, same idempotency.

### 3. Acceptance semantics

On `AcceptInvitation(token, acting_user)`:

1. Load `Invitation` by `token`.
2. Validate the reconciliation rule (§1). On failure: typed error.
3. Look up existing Membership for `(acting_user.id, invitation.store_id)`:
   - **If exists:** mark Invitation `Accepted` (record `accepted_at`, `accepted_by_user_id`, `created_membership_id ← existing.id`). Return the existing Membership. Idempotent — no duplicate created.
   - **If absent:** create new Membership with role from invitation; mark Invitation `Accepted`; return new Membership.
4. Both branches emit `InvitationAccepted` domain event. The "absent" branch also emits `MembershipCreated`.

The Invitation state change and Membership creation succeed or fail together (atomic transaction within the Tenancy context).

### 4. Decline and revoke

- **`DeclineInvitation(token, acting_user)`** requires the same email/verified/active check (so a random User cannot decline another's invitation by guessing tokens). Transitions to `Declined`.
- **`RevokeInvitation(invitation_id, acting_user)`** is issuer-side — invoked by the inviter or another actor with the appropriate capability (per ADR-0002). Does **not** require recipient match. Transitions to `Revoked`.

### 5. Expiry

An Invitation with `expires_at <= now` is treated as `Expired` for acceptance and decline purposes. The stored `state` may also be updated to `Expired` by a background job for tidiness; this is operational. Expired Invitations cannot be Accepted or Declined; the issuer must re-issue a new Invitation.

### 6. Multiple pending Invitations for the same email

Independent. Each Invitation has its own lifecycle. Accepting one Invitation does not affect any other Pending Invitations for the same email.

### 7. Already a Member

Idempotent (see §3). If the User is already an Admin of the target store, AcceptInvitation marks the Invitation `Accepted` and returns the pre-existing Membership without creating a duplicate.

### 8. Scope: Admin invitations only

The Invitation flow creates **Admin Memberships only**. Owner is set at store creation; ownership transfer uses the separate mechanism in CLAUDE.md §6.6 — **not** the invitation flow. Future roles introduced via §6.5 can opt into the invitation flow when added.

### Invitation entity attributes (canonical)

| Field | Type / value |
|---|---|
| `id` | Internal opaque |
| `store_id` | Target Store reference |
| `email` | Intended recipient (case-insensitive lookup; not LocalizedText) |
| `intended_role` | Currently always `Admin` |
| `issuer_user_id` | User who sent it |
| `issued_at`, `expires_at` | Timestamps |
| `token` | Opaque, non-guessable, single-use index value |
| `state` | `Pending` \| `Accepted` \| `Declined` \| `Revoked` \| `Expired` |
| `accepted_at`, `accepted_by_user_id`, `created_membership_id` | Nullable; set on Accept |
| `declined_at` | Nullable; set on Decline |
| `revoked_at`, `revoked_by_user_id` | Nullable; set on Revoke |

## Consequences

What this commits to:

- The reconciliation rule is fully decidable from the User and Invitation state alone. No out-of-band signals required.
- Idempotency is guaranteed by the state machine; double-clicks and retries are safe.
- Email mismatches are a typed error surface — never a silent pass-through.
- Tenancy exposes a query for "list Invitations Pending for `email`" used by the dashboard discovery path.
- Accept / Decline / Revoke each emit dedicated domain events; Audit (Gap 16) ingests them.
- The Invitation lifecycle states declared in CLAUDE.md §6.2 (Pending, Accepted, Declined, Revoked, Expired) are unchanged.

What this defers:

- **Re-invitation after Decline / Expiry / Revoke** — the issuer must explicitly re-create. No auto-reissue.
- **Bulk invitation flows** — UX feature.
- **Invitation reminders / nudges** — notification concern.
- **Invitation analytics** (acceptance rates, time-to-accept) — read-side concern.
- **Ownership transfer** — separate mechanism per §6.6.
- **Inviting recipients whose preferred contact email differs from their IdP email** — out of scope; sender must invite the IdP-verified email.

What this makes harder:

- **Email aliasing.** A recipient with `firstname.lastname@gmail.com` configured at their IdP cannot accept an invitation sent to `firstnamelastname@gmail.com` even though Gmail treats them equivalently. The match is strict case-insensitive string equality.
- **Cross-IdP-email scenarios.** A recipient cannot accept an invitation to email X if their IdP-verified email is Y, even if they own both. Workaround: issuer re-invites Y.

## References

- [gaps/resolved/12-invitation-account-reconciliation.md](../gaps/resolved/12-invitation-account-reconciliation.md)
- [ADR-0009](0009-identity-and-external-idp.md), [ADR-0002](0002-authorization-tiers-and-matrix.md)
- CLAUDE.md §6 (Invitation & Role Model); §6.3 and §6.4 updated to reflect this ADR
