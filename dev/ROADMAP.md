# gate-net - Roadmap

> Path from scaffold to a stable 1.0. Hard parts are front-loaded; each phase has hard exit criteria.
> Design and decisions: [`PLAN.md`](./PLAN.md).
>
> **Anti-deferral rule:** no listed hard task moves to a later phase unless this file records the move and the reason.

## v0.1.0 - Scaffold (DONE, 2026-10-13)
- [x] Manifest, README, CHANGELOG, REPS, dual license, CI, deny, clippy, rustfmt, DIRECTIVES, ROADMAP, PLAN.

## v0.2.0 - Design lock
Exit: the credential, session, and flow APIs and the store traits written in `docs/API.md` and approved.
- [ ] `authenticate`, `AuthOutcome`, sessions, API keys, single-use tokens, store traits, events.
- [ ] Confirm `crypt-io` provides Argon2id, keyed BLAKE3, HMAC-SHA-256, and HMAC-SHA-1 (add what is missing there).
- [ ] Open decisions O-1 to O-4 answered.

## v0.3.0 - Passwords and sessions
Exit: a login with password issues a session that validates, rotates, expires, and revokes correctly; failures are indistinguishable in outcome and timing.
- [ ] Password hashing on a bounded pool; rehash on parameter change; normalization; pepper.
- [ ] Session issue, hashed storage, idle and absolute expiry, rotation, revocation, cache with epoch.
- [ ] Uniform failure path with dummy hashing; timing tests.
- [ ] In-memory reference stores.

## v0.4.0 - Keys, factors, tokens
Exit: API keys, TOTP, recovery codes, and single-use tokens pass their vectors and abuse tests.
- [ ] API keys (keyed hash, prefix, scopes, expiry).
- [ ] TOTP (RFC 6238 vectors, replay refusal), recovery codes.
- [ ] Single-use tokens with purpose binding; reset revokes all sessions.

## v0.5.0 - Abuse resistance
Exit: credential stuffing and brute force are throttled without letting an attacker lock a real user out.
- [ ] Per-account and per-prefix budgets, progressive delay.
- [ ] Breached-password list check (offline).
- [ ] Revocations to bouncer-io; events to sentinel-sec and audit.
- [ ] Revoked-token reuse signal.

## v0.9.0 - Hardening
- [ ] Adversarial review with findings fixed test-first; property tests for every invariant; soak.

## v1.0.0 - Stable
- [ ] Frozen after the web stack has used it end to end. Passkeys follow as 1.x.
