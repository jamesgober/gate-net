# gate-net &mdash; Engineering Directives

> Engineering standards and the definition of done for this project. Read alongside `REPS.md` (root, authoritative), [`PLAN.md`](./PLAN.md) (design and decisions), and [`ROADMAP.md`](./ROADMAP.md) (current phase). If anything here conflicts with `REPS.md`, `REPS.md` wins, except where `PLAN.md` records an approved exception.

---

## 0. Philosophy

This library is built and maintained to a production standard and treated as a flagship piece of work. Plan the full path, then build one verified step at a time. "Good enough" is treated as a defect. It is a member of the owner's `-net` domain and part of the service security stack (net-io, bouncer-io, customs-sec, gate-net, gate-sec, shield-sec, sentinel-sec, sandbox-sec). A security component that cannot name the attack it stops, and prove it with a test, does not ship.

---

## 1. What this is

gate-net is authentication and sessions: passwords, API keys, session tokens, TOTP, recovery codes, single-use tokens, the login flow, and throttling. Protocol-free and storage-free (store traits). Authorization is gate-sec; user records are the users core; revocation enforcement in flight is bouncer-io. See `PLAN.md` sections 1 to 4.

---

## 2. Engineering law (non-negotiable)

- **Ours, not rented.** No third-party runtime dependency, except audited cryptographic primitives reached through `crypt-io`. First-party crates are wired when first used, at published versions (no `path` dependencies across repositories). Dev-only test tooling (`criterion`, `proptest`) is allowed.
- **Protocol-free.** Nothing in this crate knows the wire format of the traffic it protects; adapters at the top translate.
- **Unsafe.** `#![forbid(unsafe_code)]`. This crate has no reason to touch raw memory or the OS directly.
- **Hostile-input hardening.** Every input may be adversarial: explicit budgets for bytes, time, and counts, checked before the resource is committed; no reachable panics; no quadratic paths.
- **Fail closed.** When a check cannot run, the answer is no.
- **Secrets.** Compared in constant time, zeroized after use, never in logs, errors, or `Debug` output.
- **Performance.** Peak is the baseline: allocation-free hot paths, lock-free reads where correctness can be proven, no "faster" claim without `criterion` numbers at realistic scale.
- **Cross-platform.** Linux, macOS, and Windows are first-class, verified by CI on stable and MSRV 1.85. Platform differences are documented per feature.
- **Error handling.** Every fallible path returns `Result`; errors are `#[non_exhaustive]` and say what to do.
- **Production-ready.** `#![deny(missing_docs)]`; no `unwrap`/`expect`/`todo!`/`dbg!`/printing in library code; every public item has rustdoc with a runnable example; every claim in the docs maps to a test.

---

## 3. Definition of done

1. Compiles clean on Linux/macOS/Windows, stable and MSRV 1.85.
2. `fmt`, `clippy -D warnings` (all targets, all/no-default features), `test --all-features`, `cargo doc -D warnings` clean.
3. `cargo audit` + `cargo deny check` pass.
4. No `unwrap`/`expect`/`todo!`/`dbg!` in shipping code.
5. A Tier-1 API exists and headlines the docs.
6. Property tests cover every section-4 invariant; a fuzz target exists for every parser or decoder; every threat listed in `PLAN.md` has an attack test that must fail.
7. Hot-path changes carry benchmarks; no regression over 5%.
8. `README.md`, `docs/API.md`, and `CHANGELOG.md` updated; the matching `docs/release/vX.Y.Z.md` written before the tag.
9. `PLAN.md` decision log updated for every design decision made during the work.
10. v1.0.0 only after the web stack has used the API end to end, and after an adversarial security review with every finding fixed test-first.

---

## 4. Project-specific invariants

- Every secret comparison is constant time.
- No secret or token is stored in plaintext; tokens are stored only as keyed hashes.
- No secret appears in logs, errors, or `Debug` output.
- Every login failure returns the same outcome and takes the same time, whatever the reason.
- A revoked or expired session is never accepted, including from cache.
- A one-time code is accepted at most once.
- A single-use token is accepted at most once and only for its purpose.

---

## 5. Writing style

- Plain, specific, human prose in every shipping file. No marketing voice.
- Banned in shipping files: "comprehensive", "robust", "seamless", "leverage", and the em-dash character (U+2014) in prose.
- Commits: imperative present tense, descriptive subject line.
