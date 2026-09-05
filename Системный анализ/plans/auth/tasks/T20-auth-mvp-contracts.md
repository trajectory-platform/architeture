# T20 — Auth MVP contract evolution

**Size:** L · **Depends on:** T02 · **Phase:** 5

## Goal

Evolve versioned contracts from the delivered T02 baseline to the complete Auth MVP in ADR-007. Preserve published `auth.v1` compatibility; introduce `auth.v2` for any breaking wire change.

## Scope

In — `contracts` repository first:

- Access messages: repeated `roles`, repeated `permissions`, `identity_type`, `session_id`, `email_confirmed`, `active_cabinet`.
- Account statuses: `PENDING_EMAIL`, `ACTIVE`, `BLOCKED`, `DELETION_SCHEDULED`, `DELETED`.
- RPC groups from [auth.md](../../../services/auth.md): email confirmation, access context/active cabinet, sessions, OAuth connections, password/email changes, invitations, learning roles, admin roles/permissions, lifecycle, identity search/read and read-only impersonation.
- Separate actor context for privileged operations. Never trust actor identity or permissions supplied as arbitrary body fields; carry authenticated context through trusted metadata/interceptor contracts.
- Events with `event_id`, `occurred_at`, schema version and identity key. Delivery events contain `delivery_request_id`, never raw tokens or URLs.
- Internal `GetDeliveryPayload` contract restricted to Notification Service workload identity.
- Reserve removed field numbers and enum values. Document `v1` to `v2` mapping if a new package is required.

## Acceptance criteria

- [ ] `buf lint`, `buf breaking --against` and `buf generate` pass in `contracts`.
- [ ] Generated Go code compiles and `go test ./...` passes in `contracts`.
- [ ] No published `v1` field number or meaning changes in place.
- [ ] Every secret-bearing input is marked sensitive in comments; no event message can carry a raw action/refresh/OAuth token.
- [ ] Contract tests prove repeated roles/permissions, all statuses and every provider round-trip.
