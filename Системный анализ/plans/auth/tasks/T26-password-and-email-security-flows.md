# T26 — Password reset, password change and email change

**Size:** L · **Depends on:** T07, T16, T20–T24, T29 · **Phase:** 7

## Goal

Complete password recovery and account security flows, including first password for OAuth-only identities and confirmed email change.

## Scope

In:

- Password policy: 8–128 Unicode characters, at least one letter and digit, not equal to normalized email, deny common-password list, enforce a 1,024-byte UTF-8 limit before Argon2id.
- `RequestPasswordReset`: uniform response, 1-hour hashed token, maximum 3 requests/hour and 5/day per normalized address, no delivery for blocked accounts.
- `ResetPassword`: maximum 5 attempts/request; consume token, set credential, revoke every session, audit and outbox atomically.
- `ChangePassword`: require current password; replace hash and revoke all sessions except current.
- Forced reset with `auth_credential_reset`: only password identities, revoke all sessions immediately, issue a 24-hour set-password request and require an audit reason.
- Email change: current password, notify old address, confirm new address within 24 hours, support cancellation, maximum 3 attempts/day.
- Confirmed change atomically updates unique email, revokes every session including current, audits and emits security/delivery events.
- Password reset delivery uses protected `GetPasswordResetDelivery`; other action delivery uses `GetDeliveryPayload`. Kafka carries no secret.

## Acceptance criteria

- [ ] Unknown, blocked and eligible reset requests return indistinguishable public responses.
- [ ] Reset token is single-use, hashed and expires at 1 hour; attempt/request limits hold under concurrency.
- [ ] OAuth-only identity can set its first password through verified recovery.
- [ ] Password/email changes revoke exactly the required sessions.
- [ ] Forced reset rejects OAuth-only identities and its 24-hour link differs from the one-hour self-service recovery link.
- [ ] Email changes only after new-address confirmation; collisions are race-safe.
- [ ] Password, hashes, raw tokens and emails are absent from logs and event payloads where prohibited.
