# T23 — Registration, email confirmation and invitations

**Size:** L · **Depends on:** T11, T16, T20–T22, T29 · **Phase:** 6

## Goal

Evolve registration to `PENDING_EMAIL` and implement confirmation plus staff/student invitation flows through Notification Service.

## Scope

In:

- Self-register roles: `student`, `teacher`, `representative`; reject staff/admin roles.
- Validate required legal-consent flags from the versioned contract.
- Create identity, credential, initial role, 24-hour confirmation request, audit row and outbox events in one transaction.
- Allow login before confirmation but return restricted access context.
- Idempotent `ConfirmEmail`; resend no more than once per 60 seconds.
- Staff invitation: unique email, display name, at least one admin role, 72-hour set-password token.
- Representative-driven student invitation: create `student` identity and delivery request; Core Education owns the representative relationship and orchestrates its own atomic state through saga/idempotent compensation.
- `user.registered` carries learning roles and consent metadata needed by Core Education; a teacher registration causes Core Education to create the `draft` teacher application idempotently.
- Delivery events contain only `delivery_request_id`; protected delivery RPC returns the one-time URL data to Notification Service.

## Acceptance criteria

- [ ] Registration commits all Auth state and events atomically; Kafka/Notification outage leaves retriable outbox rows.
- [ ] Unconfirmed user can only obtain confirmation-screen access context.
- [ ] Confirmation token is hashed, expires after 24 hours and is single-use with safe repeat behavior.
- [ ] Staff invitation expires after 72 hours and cannot create a mixed user/staff identity.
- [ ] Teacher registration produces enough safe event data for one idempotent `draft` application in Core Education.
- [ ] Duplicate email and resend responses do not expose account existence beyond the explicit registration collision contract.
- [ ] No raw token or URL appears in Kafka, logs, traces or DLQ payload fixtures.

## Релизная поставка и recovery

Часть confirmation/self-service reset и необходимые forward-only migrations поставляются в MVP 1.0 по [09 — Релизная готовность](../../../architecture/09-release-readiness.md); полный остаток задачи закрывается позже по матрице. Hash action token и AEAD delivery payload хранятся раздельно по [ADR-008](../../../adr/ADR-008-durable-delivery-and-recovery.md). Требуется проверка рестарта между commit и отправкой, повторной выдачи того же действующего URL и очистки по CompleteDelivery/consumption/expiry. Готовность одной части не отмечает всю задачу DONE.
