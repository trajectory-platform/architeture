# T28 — Account lifecycle and anonymization

**Size:** L · **Depends on:** T20–T24, T29 · **Phase:** 7

## Goal

Implement administrative identity reads, read-only impersonation, blocking, scheduled deletion, cancellation and irreversible anonymization.

## Scope

In:

- State machine for `PENDING_EMAIL`, `ACTIVE`, `BLOCKED`, `DELETION_SCHEDULED`, `DELETED`.
- Privileged block/unblock and schedule/cancel APIs with permission, actor and mandatory reason.
- Search/read Auth-owned identity fields for Gateway aggregation; export requires `auth_user_export` and audits filters/row count.
- Read-only impersonation JWT: `impersonator_id`, `subject_id`, 30-minute TTL, no refresh/session, no room access and no mutations. Audit start/end/expiry with reason.
- Block immediately revokes all sessions and emits `identity.status_changed`.
- Scheduled worker anonymizes due identities and `PENDING_EMAIL` older than 30 days.
- Schedule deletion exactly 30 days after accepted request. Require trusted preflight proof that Billing/Core Education found no blocking balance, paid package or open financial case.
- Anonymization removes email, staff display name, credentials, OAuth connections, action requests and sessions; keeps non-personal technical identity reference.
- Remove `audit_identity_links` mapping so immutable audit rows become unlinkable without updating them.
- Protect last active `superadmin` in the same transaction.
- Coordinate downstream erasure through idempotent Kafka events; each service erases data it owns.

## Acceptance criteria

- [ ] State transition table rejects invalid and repeated destructive transitions safely.
- [ ] Blocked identity cannot login or refresh and receives no enumeration detail.
- [ ] Due deletion and 30-day pending cleanup are idempotent across concurrent workers.
- [ ] Anonymized email can be registered again; old credentials/tokens/connections cannot authenticate.
- [ ] Audit rows remain unchanged while identity mapping becomes unavailable.
- [ ] Last active `superadmin` cannot be blocked or anonymized under race.
- [ ] Impersonation expires at 30 minutes, never creates a user session and cannot authorize any mutation or room join.
