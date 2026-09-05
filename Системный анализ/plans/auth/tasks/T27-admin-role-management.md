# T27 — Administrative roles and permissions

**Size:** L · **Depends on:** T20–T22, T29 · **Phase:** 7

## Goal

Implement configurable admin roles, staff assignments and protected invariants from the business analysis.

## Scope

In:

- List permission catalog and admin roles.
- Create, update, duplicate and delete non-system roles. Validate unique code/name, 2–40 character immutable code, 2–60 character name, description up to 500 characters and at least one permission.
- Reject `*_edit` without matching `*_view`.
- Seed immutable `superadmin` with every permission and predefined `manager` per source catalog.
- Assign one or more roles to staff; effective permissions are their union.
- Implement trusted grant/revoke of learning roles. Never revoke `student` independently; revoke `representative` after Core Education proves the last dependent link was removed.
- Reject deletion of assigned role, removal of the last staff role and any mixed staff/learning assignment.
- Serialize checks protecting the last active `superadmin`.
- Require `auth_permission_view`/`auth_permission_edit`; every mutation and denial is audited.

## Acceptance criteria

- [ ] Role CRUD and assignment behavior matches RP-10–RP-19 and remains race-safe.
- [ ] Two concurrent attempts cannot remove/block the last active `superadmin`.
- [ ] Adding a new permission automatically expands `superadmin` without manual role edits.
- [ ] Effective permissions are deterministic and reflected after refresh/init.
- [ ] Role assigned to staff cannot be deleted; response identifies assignment count without leaking unrelated data.
- [ ] All mutations include actor roles, before/after and result in audit.
