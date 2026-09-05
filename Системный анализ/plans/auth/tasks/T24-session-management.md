# T24 — Session management and device metadata

**Size:** L · **Depends on:** T13, T14, T20–T22, T29 · **Phase:** 6

## Goal

Make refresh families first-class sessions with device metadata, 30-day inactivity expiry, a strict 10-session limit and user/admin revocation APIs.

## Scope

In:

- Create session atomically with first refresh token on password or OAuth login.
- Store raw user-agent, last IP, login method, created/last-active times and active cabinet. Never derive or store GeoIP.
- Persist active cabinet through the access-context API without creating a new login.
- Refresh updates last activity/IP and extends inactivity expiry to 30 days.
- Serialize concurrent logins per identity. Creating session 11 revokes the least recently active session and all its refresh tokens.
- List own sessions; revoke one non-current session; revoke all other sessions.
- `auth_session_view` and `auth_session_revoke` operations for staff; reason required and audited.
- Preserve refresh rotation, reuse detection and whole-family revocation invariants.
- Detect first login from a new stable device fingerprint and publish one idempotent `identity.new_device_login` event.

## Acceptance criteria

- [ ] 20 concurrent logins leave exactly 10 active sessions and revoke the oldest eligible sessions deterministically.
- [ ] Session data reports device/browser source fields, IP, login method and timestamps; no GeoIP exists.
- [ ] Current-session protection applies to self-service revoke but not authorized security/lifecycle revocation.
- [ ] 30 days without refresh expires a session; refresh before expiry extends it.
- [ ] Reuse revokes the whole session family and publishes one idempotent security event.
- [ ] New-device detection creates one notification event without storing GeoIP.
- [ ] Admin revocation without reason or permission fails and is audited.
