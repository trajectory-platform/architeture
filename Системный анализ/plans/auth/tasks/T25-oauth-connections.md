# T25 — OAuth login and account connections

**Size:** L · **Depends on:** T20–T24, T29 · **Phase:** 7

## Goal

Implement login, linking and unlinking for Яндекс ID, VK ID and Google without silent email-based account linking.

## Scope

In:

- Provider adapters behind one consumer-defined interface. Validate state, code exchange, issuer, audience, nonce and provider signature.
- Identify external accounts by `(provider, subject)`; enforce global uniqueness of that pair.
- Create ACTIVE user identity when provider asserts a verified email and no platform identity conflicts.
- On matching existing email, require current password or one-time email proof before linking. Return enumeration-safe responses.
- Link/unlink only for current user. Never permit OAuth for staff identity.
- Reject unlinking the final login method. Offer set-password flow for OAuth-only accounts.
- Link/unlink revokes all other sessions, writes audit and publishes `identity.security_changed` in one transaction.
- Do not persist provider tokens unless a later provider API explicitly requires them; never log them.

## Acceptance criteria

- [ ] All three providers pass adapter contract tests plus one sandbox/manual integration checklist.
- [ ] Matching email alone never links or logs in to an existing account.
- [ ] One provider subject cannot attach to two identities under concurrency.
- [ ] Last login method cannot be removed.
- [ ] Staff OAuth attempts fail before provider linking.
- [ ] Error text/timing does not reveal whether an email or provider connection exists.
