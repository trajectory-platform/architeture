# T29 — Auth audit log

**Size:** M · **Depends on:** T20, T21 · **Phase:** 6

## Goal

Create an immutable, service-owned audit trail for Auth actions without introducing a shared database or distributed transaction.

## Scope

In:

- Append-only repository for `auth_audit_log`; application DB role has INSERT/SELECT but no UPDATE/DELETE.
- Record event type, actor audit ref, actor roles, object type/ref, reason, before/after, result, request id, IP and timestamp.
- Write successful mutation audit inside the domain transaction. Write denied permission/business attempts in their own transaction.
- Require audit success for privileged Auth mutations; rollback domain change if audit insert fails.
- Pseudonymous `audit_ref` mapping supports anonymization by deleting the mapping, not mutating audit rows.
- Query/export APIs guarded by `auth_audit_view` and `auth_audit_export`; export audits itself.
- Keep records online for 12 months, then archive through an append-preserving retention job.
- Explicit redaction allowlist. Never serialize passwords, hashes, raw tokens, OAuth tokens, KEK, private keys or delivery URLs.
- Publish sanitized audit projection events only if the cross-service admin read model needs them.

## Acceptance criteria

- [ ] Domain mutation cannot commit when mandatory audit insert fails.
- [ ] Denied operation records result and reason without secret input.
- [ ] DB role cannot update/delete audit rows.
- [ ] Anonymization removes identity resolution while audit row bytes remain unchanged.
- [ ] Export requires permission and creates `audit_exported`.
- [ ] Property/redaction tests prove forbidden fields never enter JSON states or Kafka payloads.
