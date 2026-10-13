# gate-net - Project Plan

> The design, strategy, and decision record for `gate-net`. A living document: every design decision, idea, or open question about gate-net is written here or in [`ROADMAP.md`](./ROADMAP.md) before it is acted on.
>
> **Read order:** [`../REPS.md`](../REPS.md) &rarr; [`DIRECTIVES.md`](./DIRECTIVES.md) &rarr; **this file** &rarr; [`ROADMAP.md`](./ROADMAP.md).
>
> **Status:** scaffold, `0.1.0`. Design lock is `0.2.0`.

---

## Contents

1. [Purpose](#1-purpose)
2. [The gate family](#2-the-gate-family)
3. [Where it sits](#3-where-it-sits)
4. [Scope](#4-scope)
5. [Credentials](#5-credentials)
6. [Sessions](#6-sessions)
7. [Login flow](#7-login-flow)
8. [Multi-factor](#8-multi-factor)
9. [Single-use tokens](#9-single-use-tokens)
10. [Throttling and abuse](#10-throttling-and-abuse)
11. [Threats](#11-threats)
12. [Architecture](#12-architecture)
13. [Ecosystem and dependencies](#13-ecosystem-and-dependencies)
14. [Testing](#14-testing)
15. [Open decisions](#15-open-decisions)
16. [Ideas parking lot](#16-ideas-parking-lot)
17. [Decision log](#17-decision-log)

---

## 1. Purpose

gate-net answers one question: **who is this?** Credentials go in, and an authenticated identity (with a session) or a refusal comes out. Depending on configuration, it rejects a wrong login, an expired or revoked session, or a request without the factor it needs, before anything behind it runs.

---

## 2. The gate family

| Crate | Question | Layer |
|---|---|---|
| **gate-net** | who are you? (authentication, sessions) | the gate on the network path |
| `gate-sec` | may you do this? (authorization, permissions) | the security gate on each action |
| users core (name open) | the user records themselves (from a database) | consumes and bridges both |

gate-net never decides permissions, and never stores user records: it works through store traits that the users core implements.

---

## 3. Where it sits

```
parse ─▶ customs (head) ─▶ [gate-net] ─▶ bouncer (identity checkpoint) ─▶ gate-sec ─▶ customs (body) ─▶ handler
```

It is **protocol-free**. Adapters at the top find the credentials: an HTTP adapter reads a cookie or an `Authorization` header, a database wire adapter reads a handshake field, a WebSocket adapter reads the first message. All of them call the same functions.

---

## 4. Scope

### 4.1 In scope

Passwords, API keys, session tokens, TOTP and recovery codes, single-use tokens (verification, reset, magic links), the login flow with uniform responses, login throttling, session issue, rotation, expiry, revocation, the session-bound anti-forgery token primitive, events for audit and detection, and store traits.

### 4.2 Out of scope, and where it goes

| Not here | Where |
|---|---|
| permissions and policy | `gate-sec` |
| user records, profiles, account lifecycle | users core |
| sending email or SMS (verification, reset) | an SMTP crate (not built yet); gate-net produces the token, the app sends it |
| cookies, headers, CSRF wiring for HTTP | the web layer's adapter |
| enforcing revocation on in-flight requests | `bouncer-io` (gate-net issues the revocation) |
| the hash and cipher primitives | `crypt-io` |

---

## 5. Credentials

| Credential | Design |
|---|---|
| **password** | Argon2id (RFC 9106) through `crypt-io`; parameters stored with each hash so they can be raised later; rehash on the next successful login when parameters change; optional server-side pepper held in `key-vault`; Unicode normalized (NFKC) before hashing so the same password typed on two keyboards matches; length bounded (minimum per policy, maximum 1,024 bytes so hashing cost cannot be abused) |
| **password hashing pool** | Argon2id is memory-hard on purpose, so a login flood is a memory flood: hashing runs on a bounded worker pool, behind the throttles in section 10 |
| **API key** | high-entropy random keys; stored as a keyed fast hash (BLAKE3 keyed or HMAC-SHA-256 with a key from `key-vault`), not Argon2id, since slow hashing protects low-entropy human secrets and only costs us here; a short public prefix for lookup and for recognizing leaked keys; scopes and expiry; constant-time comparison |
| **session token** | see section 6 |
| **TOTP** | see section 8 |
| **recovery codes** | random, single use, stored hashed like passwords |
| **client certificate identity** (later) | from the TLS adapter's verified peer certificate |
| **passkeys / WebAuthn** (later) | needs CBOR, COSE, and ECDSA P-256 verification; planned after the core |

Every secret is compared in constant time and zeroized after use.

---

## 6. Sessions

| Aspect | Design |
|---|---|
| token | 256 bits from the OS CSPRNG (`mod-rand`), encoded for transport by the adapter |
| storage | only a keyed hash of the token is stored; a database leak does not leak live sessions |
| expiry | idle timeout and absolute lifetime, both configurable; whichever comes first ends the session |
| rotation | a new token on login, on privilege change, and periodically; the old token is invalidated |
| context | the address prefix and client fingerprint at issue are recorded; a change raises a signal and can require re-authentication |
| revocation | single session, all sessions of an account, all sessions everywhere; every revocation is passed to `bouncer-io`, which reaches requests already in flight |
| revoked-token reuse | presenting a revoked or expired token is reported as a high-confidence hijack signal |
| cache | validated sessions are cached (`cache-mod`) with the revocation epoch, so a revocation invalidates the cache entry immediately |
| anti-forgery primitive | a secondary token bound to the session, for the web layer's CSRF defence |

---

## 7. Login flow

```
authenticate(credentials, context) -> AuthOutcome
AuthOutcome = Granted(Session) | NeedFactor(Challenge) | Rejected
```

- **No account enumeration:** an unknown account, a wrong password, and a locked account return the same `Rejected`, take the same time (a dummy hash is computed for unknown accounts), and produce the same public message. The real reason goes only to events.
- **Step by step:** first factor, then `NeedFactor` if the account or the policy requires a second, then `Granted`.
- **Events:** success, failure, factor required, factor failed, session issued, rotated, revoked.

---

## 8. Multi-factor

| Factor | Design |
|---|---|
| TOTP (RFC 6238) | 30-second steps, 6 digits, a window of one step either side; each code accepted once (replay refused); secrets stored encrypted through `key-vault`; HMAC-SHA-1 by default for authenticator-app compatibility, SHA-256 optional |
| recovery codes | a set issued at enrolment, each single use, hashed at rest |
| step-up | `gate-sec` can require that the second factor was proven within a recent window for sensitive actions; gate-net records when it was |
| passkeys (later) | WebAuthn registration and assertion |

---

## 9. Single-use tokens

For email verification, password reset, invitations, and magic links:

- random 256-bit tokens, stored hashed,
- short expiry, single use, bound to their purpose (a verification token cannot reset a password),
- issuing a new one invalidates the previous one of the same purpose,
- a password reset revokes every session of the account.

---

## 10. Throttling and abuse

| Defence | Detail |
|---|---|
| per-account budget | failed attempts per account per window, then progressive delay; never a hard lock that an attacker could use to lock real users out |
| per-prefix budget | failed attempts per address prefix |
| global hashing budget | the hashing pool bounds concurrent work |
| breached passwords | an offline list check at set time (no network call) |
| distributed stuffing | failures across many accounts from related sources are reported to `sentinel-sec`, which sees across accounts |
| strikes | repeated failures become strikes at `bouncer-io` |

---

## 11. Threats

| Threat | Defence |
|---|---|
| credential stuffing, brute force | budgets, progressive delay, hashing pool, sentinel correlation |
| account enumeration | uniform responses and timing |
| password database theft | Argon2id with stored parameters, optional pepper outside the database |
| session theft | hashed storage, rotation, context binding, short lifetimes, revoked-token reuse detection |
| session fixation | a new token on every login |
| replayed one-time codes | each TOTP code accepted once |
| leaked API keys | recognizable prefix (secret scanners can find them), hashed storage, scopes, expiry, instant revocation |
| timing attacks | constant-time comparison everywhere |
| secrets left in memory | zeroization after use |
| reset-token abuse | purpose binding, single use, short expiry, session revocation on reset |

---

## 12. Architecture

```
src/
  lib.rs         crate docs, Tier-1 entry points
  identity.rs    PrincipalId (opaque), AuthContext
  password.rs    hashing, verification, rehash, normalization, policy
  apikey.rs      generation, keyed hashing, prefix, scopes
  session.rs     issue, validate, rotate, expire, revoke
  totp.rs        TOTP and recovery codes
  token.rs       single-use tokens
  flow.rs        authenticate(), AuthOutcome
  throttle.rs    budgets and delays
  store.rs       CredentialStore and SessionStore traits; in-memory reference stores for tests
  events.rs      event types
  error.rs       GateError (#[non_exhaustive])
```

Store traits keep gate-net free of any database. The users core implements them over Hive DB (or anything else); the in-memory reference stores make every test runnable without one.

---

## 13. Ecosystem and dependencies

| Crate | For |
|---|---|
| `crypt-io` | Argon2id, BLAKE3 keyed hashing, HMAC (crypto primitives are an accepted exception: they come from audited implementations) |
| `key-vault` | pepper, TOTP secret encryption keys, API key hashing key |
| `mod-rand` | token generation from the OS CSPRNG |
| `clock-lib` | expiry and TOTP time, mockable in tests |
| `cache-mod` | session cache |
| `bouncer-io` | receives session and account revocations (through a trait, to keep the dependency one way) |
| `sentinel-sec`, `audit-trail` | receive events |

Gap to check at design lock: whether `crypt-io` exposes HMAC-SHA-1 for standard TOTP. If not, it is added there, not here.

---

## 14. Testing

- RFC 6238 test vectors for TOTP; RFC 9106 vectors for Argon2id parameters through crypt-io.
- Timing tests: unknown account and wrong password indistinguishable within a tight bound.
- Session lifecycle: every expiry, rotation, and revocation path; revoked-token reuse raises its signal.
- Throttling under concurrency; no lockout of a real user by an attacker's failures alone.
- Property tests: tokens unique, never logged, never in `Debug` output.
- Benchmarks: session validation (cache hit and miss), login throughput at the configured Argon2id cost.

---

## 15. Open decisions

| Id | Question | Recommendation |
|---|---|---|
| O-1 | Argon2id parameters: start from OWASP minimums (19 MiB, t=2, p=1) and tune per server? | yes, tuned on the production hardware |
| O-2 | Self-encoded (signed) session tokens, or opaque tokens with server-side lookup? | opaque: revocation is instant and nothing sensitive travels |
| O-3 | Passkeys in 1.0, or after? | after 1.0; the core first |
| O-4 | Publish `0.1.0` to crates.io to claim the name? | yes |

---

## 16. Ideas parking lot

- Device registry: named, revocable devices per account.
- Login notifications through the app (gate-net emits the event; the app decides).
- Delegated sessions for support staff with full audit and the user's consent.

---

## 17. Decision log

| Id | Date | Decision | Reason |
|---|---|---|---|
| D-001 | 2026-10-13 | `gate-net` handles authentication and sessions; `gate-sec` handles authorization | James's split: the network gate is who you are, the security gate is what you may do |
| D-002 | 2026-10-13 | Protocol-free core with store traits | the same login and sessions serve web, database, and admin protocols |
| D-003 | 2026-10-13 | Argon2id for human secrets; keyed fast hash for high-entropy keys and tokens | slow hashing protects guessable secrets and only costs time on random ones |
| D-004 | 2026-10-13 | Tokens stored only as keyed hashes | a database leak must not leak live credentials |
| D-005 | 2026-10-13 | Uniform responses and timing for every login failure | no account enumeration |
| D-006 | 2026-10-13 | Revocation flows to bouncer-io | revoked sessions must stop requests already in flight |
