# T31 — Complete Auth MVP E2E suite

**Size:** L · **Depends on:** T16, T23–T30 · **Phase:** 8

## Goal

Prove the complete Auth MVP against real Postgres, Kafka and Redis plus fake deterministic OAuth/Notification peers.

## Scope

In:

- Extend T18 harness without duplicating baseline scenarios.
- Registration for all learning roles, confirmation, resend limit and restricted pre-confirmation access.
- Staff/student invitations and protected Notification delivery payload retrieval.
- Multi-role token/init, configurable staff roles, permission union and last-superadmin races.
- Session listing, metadata, 10-session eviction, inactivity expiry, self/admin revocation and refresh reuse.
- OAuth login/link/unlink for all providers, matching-email proof and last-login-method invariant.
- Password reset/change, email change, limits and exact session revocation effects.
- Block, scheduled delete, pending cleanup and anonymization with downstream Kafka events.
- Audit atomicity, denial records, redaction and unlinking after anonymization.
- Kafka outage/recovery, duplicate delivery and stable `event_id` consumer deduplication.

## Acceptance criteria

- [ ] Scenarios cover RP-1–RP-24 and SS-1–SS-26 that belong to Auth boundary.
- [ ] Concurrency cases repeat reliably with no sleeps; polling always has deadlines.
- [ ] Suite passes three consecutive CI runs and reports useful domain-state diagnostics on failure.
- [ ] Test logs, Kafka records and failure dumps contain no raw secret.
- [ ] Baseline T18 compatibility scenarios remain green throughout migration.
