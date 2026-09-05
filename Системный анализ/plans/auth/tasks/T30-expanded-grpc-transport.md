# T30 — Expanded Auth gRPC transport

**Size:** L · **Depends on:** T15, T20, T22–T29 · **Phase:** 8

## Goal

Expose the complete Auth MVP through gRPC while preserving uniform security errors, trusted actor context and secret-free telemetry.

## Scope

In:

- Implement all RPCs introduced by T20 with thin DTO mapping to use cases.
- Authenticate Gateway and Notification workload identities. Restrict `GetPasswordResetDelivery` and `GetDeliveryPayload` to Notification Service.
- Extract actor/session/request/IP/user-agent context from authenticated metadata. Ignore spoofable body fields.
- Centralize validation and business-error mapping. Keep login/recovery/linking enumeration-safe.
- Enforce byte bounds before expensive password hashing or provider calls.
- Update gRPC health readiness to include migrated DB, usable signing key and Kafka producer.
- Preserve interceptor order and never log request/response payloads.

## Acceptance criteria

- [ ] Bufconn tests cover every RPC, permission boundary and error mapping.
- [ ] Spoofed actor metadata from an untrusted caller cannot authorize an operation.
- [ ] Notification delivery RPC rejects Gateway/user credentials and accepts only Notification workload identity.
- [ ] Wrong email, wrong password, blocked account and unknown OAuth link preserve approved indistinguishable responses.
- [ ] Passwords and tokens are absent from captured logs/traces.
- [ ] Legacy clients remain supported according to T20 compatibility plan.
