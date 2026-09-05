# T21 — Identity, access and session schema evolution

**Size:** L · **Depends on:** T03, T20 · **Phase:** 5

## Goal

Add forward-only auth_db migrations for ADR-007 without editing or renumbering migrations delivered by T03.

## Scope

In:

- Add `identity_type`, lifecycle timestamps and nullable email support to `identities`.
- Add `user_identities`, `staff_identities`, `identity_learning_roles`, `permissions`, `admin_roles`, `admin_role_permissions`, `staff_role_assignments`.
- Add `sessions`, `oauth_connections`, generalized hashed `auth_action_requests`, `auth_audit_log` and `audit_identity_links`.
- Extend outbox with stable `event_id`, Kafka `topic`, `message_key` and schema version fields.
- Seed the permission catalog and deterministic system roles `superadmin` and `manager`; seed data must be idempotent.
- Backfill legacy `student`/`teacher` identities as users and legacy `admin` identities as staff. Backfill role bindings and sessions for active refresh families.
- Add DB constraints for one subtype, valid learning roles, unique provider subject, one active refresh token per family and role assignment references.
- Keep legacy `identities.role` readable during transition. Do not drop it in this task.

## Acceptance criteria

- [ ] Upgrade works from an actual T03/T01–T11 database with representative data and active refresh chains.
- [ ] Existing identities retain equivalent effective access after backfill.
- [ ] Re-running migrations and seeds is safe.
- [ ] Integration tests reject mixed user/staff subtype, duplicate provider subject and invalid role bindings.
- [ ] Rollback policy is documented; destructive down migration is not presented as safe after dual-write begins.
