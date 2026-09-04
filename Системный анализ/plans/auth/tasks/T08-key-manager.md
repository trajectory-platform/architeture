# T08 — Signing-key manager

**Size:** L · **Depends on:** T05, T06 · **Phase:** 1

## Goal

`internal/service/keys` (business logic) + `internal/infra/keycrypto`: generation, KEK encryption at rest, and the lifecycle `NEXT → ACTIVE → RETIRING → RETIRED` per [auth.md — Ротация ключей подписи](../../../services/auth.md). This is the riskiest piece — isolate it well and over-test transitions.

## Scope

In:

- **Key material**: Ed25519 (alg `EdDSA`). `kid` = random 16B base64url. Public part marshaled to JWK (`jwx`), stored in `public_jwk`; private key encrypted with AES-256-GCM under KEK from `AUTH_KEK` (nonce prepended), stored in `private_key`.
- `keycrypto` package: `Seal(priv) ([]byte, error)` / `Open(blob) (ed25519.PrivateKey, error)`. KEK only touches this package.
- `service/keys.Manager`:
  - `Bootstrap(ctx)` — on start: if no ACTIVE key exists, create one directly as ACTIVE (cold start, no caches to warm); if ACTIVE exists, ensure a NEXT exists when ACTIVE age > rotation threshold.
  - `Rotate(ctx)` — promote NEXT→ACTIVE **only if** NEXT was published ≥ JWKS cache-warm period (config `AUTH_JWKS_WARM`, default 2× gateway cache TTL); demote old ACTIVE→RETIRING with `not_after = now + access TTL + skew`; RETIRING past `not_after` → RETIRED. All via CAS `UpdateStatus` (T06) — safe with replicas.
  - `Active(ctx) (SigningKey + decrypted private, error)` — with small in-memory cache (TTL ~30s) to avoid per-issue DB hit + decryption.
  - `Publishable(ctx) ([]JWK, error)` — NEXT/ACTIVE/RETIRING public JWKs for T10.
- Background ticker started in `internal/app` (interval `AUTH_KEY_ROTATE_CHECK`, default 1h) calling `Rotate`; rotation threshold config `AUTH_KEY_MAX_AGE` (default 30d).

Out: JWKS HTTP serving (T10), JWT signing (T9), external KMS (KEK via env is the MVP per [07 — Security](../../../architecture/07-security.md)).

## Implementation notes

- Never two ACTIVE: partial unique index from T03 is the backstop; promotion order — demote old ACTIVE→RETIRING first, then NEXT→ACTIVE, both inside one tx (`WithTx`).
- Wrong/rotated KEK → `Open` fails → process must fail readiness loudly, not sign garbage.
- Clock via the `service.Clock` interface — tests use fixed clock.

## Acceptance criteria

- [ ] Unit tests (fake repos + fixed clock): bootstrap on empty DB → one ACTIVE; NEXT created at threshold; NEXT not promoted before warm period; promotion swaps statuses atomically; RETIRING→RETIRED only after `not_after`.
- [ ] Integration: two manager instances running `Rotate` concurrently → invariant holds (CAS + index), no double-ACTIVE, no lost keys.
- [ ] Seal/Open round-trip; tampered ciphertext and wrong KEK fail; KEK absent → startup fails.
- [ ] Decrypted private keys never logged, never returned by `Publishable`.
