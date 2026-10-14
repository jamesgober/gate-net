# gate-net - Session Security

> The complete design for sessions: what the client holds, what the server holds, how every request is verified, where sessions are stored, every defence with its cost, and the experiments. Companion to [`PLAN.md`](./PLAN.md) section 6.
>
> **Rule for this document:** every measure states its cost (latency, memory, or operational) so security and performance are decided together, never traded silently. Costs marked *estimate* are design targets to be confirmed by benchmarks; they are replaced with measured numbers as code lands.

---

## Contents

1. [Principles](#1-principles)
2. [Anatomy of a session](#2-anatomy-of-a-session)
3. [Lifecycle](#3-lifecycle)
4. [Verifying a request, step by step, with costs](#4-verifying-a-request-step-by-step-with-costs)
5. [Storage backends](#5-storage-backends)
6. [More than one node](#6-more-than-one-node)
7. [Threats and defences](#7-threats-and-defences)
8. [Every measure, with its cost](#8-every-measure-with-its-cost)
9. [Experiments and new ideas](#9-experiments-and-new-ideas)
10. [Profiles](#10-profiles)
11. [Open decisions](#11-open-decisions)

---

## 1. Principles

1. **Nothing the client sends is trusted.** The client holds a random token. It is a lookup key, not a claim.
2. **Every request is verified against the server's record.** No request is authenticated by the token alone.
3. **Revocation is immediate.** Logout, password change, role change, or a security decision takes effect on the next request, on every node, including requests already in flight (through bouncer-io).
4. **Speed comes from where the record is kept, not from checking less.** The server's record lives in memory in front of a durable store; the full check runs every time.
5. **Storage is pluggable.** No backend is hard-coded; memory is the default.
6. **Defence in depth.** Every layer assumes the one before it has failed.

---

## 2. Anatomy of a session

### 2.1 What the client holds

| Item | Content | Trusted? |
|---|---|---|
| session cookie | `__Host-sid`: 32 random bytes from the OS CSPRNG, base64url encoded; `Secure`, `HttpOnly`, `SameSite=Lax`, `Path=/` | no: a lookup key only |
| CSRF token (web) | a second random value bound to the session, sent in forms and headers | no: compared to the server's copy |

Nothing else. No user id, no roles, no expiry, no signature to trust.

### 2.2 What the server holds (the session record)

| Field | Purpose |
|---|---|
| `key` | keyed hash of the token (BLAKE3 keyed, server key in key-vault); the token itself is never stored |
| `account` | the principal id |
| `subject` | a snapshot of roles and attributes for gate-sec |
| `created`, `last_seen`, `idle_expiry`, `absolute_expiry` | lifetime |
| `factor_at` | when the second factor was last proven (for step-up) |
| `bindings` | address prefix, TLS client fingerprint, keyed user-agent hash (per-account secret) |
| `epoch` | the account epoch the record was validated under |
| `csrf` | the session's CSRF secret |
| `device` | the device entry this session belongs to (device list, revoke per device) |
| `flags` | elevated (sudo mode) until a time, decoy marker |

At rest in a durable backend the record is encrypted (AEAD, key in key-vault); only `key` and the expiry fields stay readable for lookup and cleanup.

---

## 3. Lifecycle

| Step | What happens |
|---|---|
| **issue** | after a successful login (and second factor if required): new random token, new record, any pre-login token discarded (fixation defence), device entry created or updated |
| **validate** | every request, section 4 |
| **touch** | `last_seen` updated at most once per interval (not every request), so a busy session does not cost a write per request |
| **rotate** | new token on login, on privilege change, on step-up, and every rotation interval; the old token becomes a tombstone |
| **elevate** | sensitive actions require a fresh second factor; the session gets a short elevated window (sudo mode) |
| **expire** | idle or absolute expiry, whichever comes first; expired records are removed by a timer wheel, not by scanning |
| **revoke** | logout, revoke device, revoke all, password reset, security decision: record removed, token tombstoned, account epoch bumped, bouncer-io told |
| **tombstone** | revoked and rotated tokens are remembered (as keyed hashes) for a window; any later use is a theft signal |

---

## 4. Verifying a request, step by step, with costs

| # | Step | On failure | Cost (estimate, one CPU core) |
|---|---|---|---|
| 1 | read the cookie (cookie-net, zero-copy) | anonymous | ~10 ns |
| 2 | decode base64url, check length | refuse | ~10 ns |
| 3 | keyed hash of the 32-byte token | | ~50 to 100 ns |
| 4 | look up the record in the in-memory store | not found: check tombstones (step 4a), then refuse | ~20 to 60 ns |
| 4a | tombstone lookup (only on a miss) | revoked-token reuse: high-confidence signal to sentinel-sec and bouncer-io | ~20 to 60 ns |
| 5 | expiry check (idle, absolute) | refuse, remove | ~2 ns |
| 6 | epoch check against the account epoch | reload from the durable store | ~1 to 5 ns (one atomic read) |
| 7 | bindings: prefix compare, fingerprint compare, keyed user-agent hash compare (constant time) | per policy: step-up, re-login, or revoke | ~100 to 200 ns (the keyed hash of the user agent dominates) |
| 8 | hand the subject to gate-sec | | ~1 ns (a reference) |
| | **total on a hit** | | **about 0.2 to 0.5 µs** |
| | **total on a miss** (record loaded from the durable store) | | the backend's read latency (in-process embedded store: microseconds; Hive DB over the network: sub-millisecond to milliseconds) |

Two optimizations keep the hit path at the low end without weakening it:

- the user-agent binding hash is cached per connection (recomputed only if the header changes on that connection),
- `last_seen` is written at most once per touch interval.

For comparison, verifying against a database on every request costs a network round trip per request (typically 0.2 to 2 ms): roughly a thousand times slower for the same check.

---

## 5. Storage backends

Sessions go through one trait, `SessionStore`. gate-net ships the default and the tiering; other backends implement the trait.

| Backend | Where it lives | Survives restart | Multi-node | Notes |
|---|---|---|---|---|
| **memory** (default) | in-process, sharded hash map | no | no (single node) | fastest; right for one server; a restart logs everyone out unless tiered |
| **tiered**: memory in front of a durable store | memory as L1, any durable backend as L2 | yes | yes, with invalidation (section 6) | the recommended production shape: memory speed, durable truth |
| **embedded database** (`emdb`, `lsm-db`) | a file next to the service | yes | no | local persistence done properly (atomic writes, crash safety, expiry) |
| **Hive DB / hive-kv** | the database | yes | yes | when the site already runs on Hive |
| **gate-db** (proposed, section 5.2) | a purpose-built auth database | yes | yes, designed for it | the long-term target |
| external stores | anything implementing the trait | depends | depends | for deployments that need it |

### 5.1 File sessions (PHP style): not offered

One file per session in a directory is the PHP default, and rarely used elsewhere. It is not forbidden by any standard, but it is a poor design: file locking serializes requests of the same user, cleanup needs scanning, permissions on shared hosts have leaked sessions between sites, and there is no way to propagate revocation. The embedded-database backend covers every legitimate reason for local file persistence (survive restart, no external service) without those problems.

### 5.2 gate-db: a purpose-built auth database (proposed)

A general key-value store can hold sessions. A database built only for authentication data can do it faster and more safely, because the workload is narrow and known:

| Property | What a purpose-built auth DB can do that a general store does not |
|---|---|
| fixed-size records keyed by a 32-byte hash | a flat open-addressing table: no tree, no variable-length keys, predictable cache behaviour |
| expiry is the main access pattern | a timer wheel for TTLs: expiry costs nothing per lookup and needs no scans |
| revocation must cross nodes fast | a priority replication lane for revocations and epoch bumps, ahead of ordinary writes |
| high-churn small writes (`last_seen`) | coalesced writes, never on the hot path |
| counters (login attempts, budgets) | atomic counters with windows built in |
| security | encryption at rest by default, secure deletion (crypto-shredding of expired sessions), tamper-evident audit (`audit-trail`), constant-time paths |
| isolation | auth data separated from application data: a compromised application database does not expose credentials |
| contents | accounts' auth fields, credential hashes, roles and permissions, sessions, tombstones, API keys, devices, MFA enrolment |
| scale | sharding by account id; read-mostly role data replicated everywhere; built for clustering from the first line |

Built from the portfolio's own storage crates (wal-db, page-db, store-io, raft-io, crypt-io, key-vault), so it is an assembly of proven parts, not storage written from zero. Home: `db-collection/gate-db` (name free on crates.io, 2026-10-13). Status: proposed, awaiting James.

---

## 6. More than one node

The in-memory store alone is single-node. With tiering it scales out:

- **Durable truth** in the shared backend (Hive DB, gate-db).
- **Each node's memory is a cache** (L1) of records it has seen.
- **Invalidation:** every revocation and epoch bump is broadcast to all nodes (through the node cores and the security core, `_strategy/CORE-ARCHITECTURE.md`); each node drops the record from L1 at once.
- **Safety net:** each L1 entry also has a short maximum age (for example 60 s), so even a lost invalidation cannot keep a revoked session alive longer than that bound.
- **Fail safe:** if a node loses its invalidation channel for longer than the bound, it stops trusting L1 and reads through to the durable store until the channel is back (slower, never wrong).
- **No sticky sessions needed:** any node can serve any request.

Cost: a broadcast per revocation (rare), an L1 miss per node per session after an invalidation, and the read-through penalty only while a node is cut off.

---

## 7. Threats and defences

| Threat | Defences |
|---|---|
| token guessing | 256 bits of randomness (a guess succeeds with probability about 2^-256); budgets on invalid tokens per prefix; misses are strikes |
| theft over the network | HTTPS only (`Secure`, HSTS from shield-sec policy), `__Host-` prefix |
| theft by script (XSS) | `HttpOnly`; content security policy; context-aware escaping in rendering |
| theft by malware on the user's device (infostealers copying cookies) | bindings (fingerprint, prefix, user agent) detect use from another machine; device-bound sessions (9.2) make a copied cookie useless; anomaly scoring in sentinel-sec |
| session fixation | a fresh token at every login; pre-login tokens discarded |
| cross-site request forgery | `SameSite=Lax` plus the session-bound CSRF token on state-changing requests |
| replay of a stolen cookie | bindings, rolling tokens (9.1), tombstone reuse detection |
| stale privileges after a role change | account epoch checked on every request |
| logout that does not log out | server-side revocation; the cookie is cleared but correctness never depends on it |
| database leak | only keyed hashes of tokens stored; records encrypted at rest |
| insider reading sessions | encryption at rest, audit trail of access, no plaintext tokens anywhere |
| timing side channels | lookup by keyed hash; constant-time comparisons |
| session flooding (creating millions of sessions) | per-account concurrent-session cap; sessions created only after authentication |
| long-lived stolen sessions | absolute expiry; rotation |
| admin session hijack | sudo mode (9.3), network rules on admin actions (gate-sec), shorter admin lifetimes |

---

## 8. Every measure, with its cost

| Measure | Default | Latency cost | Other cost |
|---|---|---|---|
| keyed token hash, in-memory lookup | on | ~0.1 µs | memory: one record per live session (about 256 to 512 bytes each, estimate) |
| expiry and epoch checks | on | ~5 ns | none |
| prefix and fingerprint binding | on | ~5 ns | users on mobile networks change prefix more often: policy decides step-up versus signal only |
| keyed user-agent binding | on | ~100 to 200 ns (cached per connection) | browser updates change it: treated as a signal plus step-up, not a hard logout |
| tombstones | on | only on a miss | memory for the tombstone window |
| CSRF token | on (web) | ~50 ns compare | none |
| rotation on login and privilege change | on | one store write at those moments | none |
| periodic rotation | on (every 15 min, estimate) | one store write and one `Set-Cookie` per interval per session | concurrent tabs handled by a grace window |
| touch coalescing | on | none (removes writes) | `last_seen` precise to the interval only |
| encryption at rest | on for durable backends | none on the hit path; ~1 µs per load or store (estimate) | key management |
| concurrent-session cap | on (configurable) | one counter check at login | none |
| tiered store with invalidation | on in production | none on hits | broadcast per revocation |
| sudo mode for sensitive actions | on | a second factor prompt for the user, minutes apart | user friction, deliberately |
| rolling tokens | experiment | one write per roll | complexity with many tabs |
| device-bound sessions | experiment | a signature check per refresh (minutes apart), ~50 to 100 µs | browser support limited |
| request signing for API clients | optional | ~10 to 50 µs per request (signature verify) | client must sign |

---

## 9. Experiments and new ideas

### 9.1 Rolling tokens with theft detection

The token is replaced at intervals; the old one stays valid for a short grace window (for tabs and in-flight requests), then becomes a tombstone. If a tombstoned token is ever presented, two copies of the session exist, so one is stolen: the whole session is revoked and the account is challenged. Cost: one write and one `Set-Cookie` per roll. Risk to manage: several tabs racing a roll (the grace window handles it).

### 9.2 Device-bound sessions

The browser generates a private key in secure hardware and proves possession to the server when refreshing the session (Device Bound Session Credentials, a new browser standard). A cookie copied by malware to another machine fails the proof. Cost: a signature verification per refresh, minutes apart. Status: support limited to some browsers; implement as an upgrade path where available.

### 9.3 Sudo mode (elevated sub-session)

Sensitive actions (changing email, password, roles, payment details, exporting data, admin actions) require a fresh second factor and grant a separate short-lived elevated state (5 to 15 minutes), carried as its own cookie scoped to the sensitive paths. A stolen ordinary session cannot perform them. Cost: user friction at sensitive moments only.

### 9.4 Decoy sessions

Tokens planted where an attacker who has read our files, logs, or backups would find them. They correspond to no real session; any use is definitive and triggers sentinel-sec's response. Cost: none at runtime beyond a set membership check on misses.

### 9.5 Behavioural continuity

sentinel-sec keeps a light profile per session (request rhythm, typical paths, client fingerprint). A sudden break in continuity (a session that browsed normally starts bulk-exporting from a new fingerprint) triggers step-up. Cost: off the request path (sentinel-sec intake).

### 9.6 Impossible travel

Two uses of one session from address prefixes too far apart for the time between them. Needs an address-to-location data source (decision open). Cost: one lookup per prefix change.

### 9.7 Proof of possession for API clients

For non-browser clients: requests signed with the client's key (HTTP Message Signatures, RFC 9421), or tokens bound to the client's TLS certificate. A stolen token alone is useless. Cost: a signature verification per request (~10 to 50 µs).

### 9.8 Split storage of the token hash key

The server key used to hash tokens is held only in key-vault (page-locked) and rotated; records carry the key id. A stolen database plus a stolen old key still cannot match current tokens after rotation. Cost: rehash on rotation, done lazily on next use.

---

## 10. Profiles

| Profile | For | Settings |
|---|---|---|
| **standard** (default) | public sites | everything in section 8 marked on; tiered store when durable backend exists |
| **strict** | admin panels, finance, health data | shorter lifetimes (idle 15 min, absolute 8 h), bindings mismatch forces re-login, sudo mode for all writes, concurrent-session cap 3, rolling tokens on |
| **paranoid** | the most sensitive deployments | strict plus device-bound sessions required where supported, proof of possession for all API clients, any binding change revokes |

---

## 11. Open decisions

| Id | Question | Recommendation |
|---|---|---|
| S-1 | Build gate-db (section 5.2)? | yes: the workload is narrow and security-critical enough to justify a purpose-built store, assembled from portfolio crates |
| S-2 | Address-to-location data source for impossible-travel checks | defer; prefix changes plus bindings first |
| S-3 | Default rotation interval | 15 minutes, tuned after measurement |
