# T07 — Password hasher (argon2id)

**Size:** S · **Depends on:** T05 · **Phase:** 1

## Goal

`internal/infra/argon2id` implementing the `service.PasswordHasher` interface: hashing, verification, and a dummy-verify path that equalizes Login timing for unknown emails ([auth.md — Login](../../../services/auth.md)).

## Scope

In:

- `Hash(password) (string, error)` — argon2id, params from config (`AUTH_ARGON2_MEMORY` default 64MiB, `ITERATIONS` default 3, `PARALLELISM` default 2, salt 16B, key 32B). Output: standard PHC string `$argon2id$v=19$m=...,t=...,p=...$salt$hash`.
- `Verify(password, phc) (bool, error)` — parses PHC, **uses params from the stored string** (not current config — old hashes stay verifiable after param bumps), `subtle.ConstantTimeCompare`.
- `DummyVerify(password)` — runs full argon2id against a fixed baked-in dummy PHC and discards the result; same params as current config so cost matches real verify.
- `NeedsRehash(phc) bool` — params weaker than current config (used by Login to opportunistically upgrade hashes; wiring in T12).
- Password policy check lives in `service` use cases (min length 8, max 72 — but enforce max ~1KB input to bound argon2 cost); this package only hashes.

Out: any storage, any timing instrumentation beyond tests.

## Acceptance criteria

- [ ] Round-trip: Hash → Verify true; wrong password → false, nil error; malformed PHC → error.
- [ ] Hash output parses with reference parser format; params encoded correctly.
- [ ] Verify uses embedded params: hash made with t=1 verifies even when config says t=3; `NeedsRehash` flags it.
- [ ] Benchmark test documents per-op cost at default params (guards accidental 10× param typo).
- [ ] No logging of inputs/outputs anywhere in package.
