<h1 align="center">
    <img width="99" alt="Rust logo" src="https://raw.githubusercontent.com/jamesgober/rust-collection/72baabd71f00e14aa9184efcb16fa3deddda3a0a/assets/rust-logo.svg">
    <br>
    <b>gate-net</b>
    <br>
    <sub><sup>AUTHENTICATION AND SESSIONS</sup></sub>
</h1>

<div align="center">
    <a href="https://crates.io/crates/gate-net"><img alt="Crates.io" src="https://img.shields.io/crates/v/gate-net"></a>
    <a href="https://crates.io/crates/gate-net"><img alt="Downloads" src="https://img.shields.io/crates/d/gate-net?color=%230099ff"></a>
    <a href="https://docs.rs/gate-net"><img alt="docs.rs" src="https://img.shields.io/docsrs/gate-net"></a>
    <a href="https://github.com/jamesgober/gate-net/actions"><img alt="CI" src="https://github.com/jamesgober/gate-net/actions/workflows/ci.yml/badge.svg"></a>
    <a href="https://github.com/rust-lang/rfcs/blob/master/text/2495-min-rust-version.md"><img alt="MSRV" src="https://img.shields.io/badge/MSRV-1.85%2B-blue"></a>
</div>

<br>

<div align="left">
    <p>
        <strong>gate-net</strong> answers one question: who is this? Credentials go in (a password, an API key, a session token, a one-time code), and an authenticated identity or a refusal comes out. It owns password hashing, API keys, session issue, rotation, expiry, and revocation, multi-factor checks, and the login throttling that makes credential stuffing expensive.
    </p>
    <p>
        It is protocol-free: an HTTP adapter finds the token in a cookie, a database wire protocol finds it in a handshake field, and both call the same functions. Every secret is compared in constant time, stored only as a hash, and zeroized after use. Authorization (what an identity may do) is the sister crate, <code>gate-sec</code>.
    </p>
    <br>
    <hr>
    <p>
        <strong>MSRV is 1.85+</strong> (Rust 2024 edition). <code>#![forbid(unsafe_code)]</code>.
    </p>
    <blockquote>
        <strong>Status: scaffold (<code>0.1.0</code>).</strong> No functional code yet. The design is written down in <a href="./dev/PLAN.md"><code>dev/PLAN.md</code></a> and the schedule in <a href="./dev/ROADMAP.md"><code>dev/ROADMAP.md</code></a>; the public API is locked in <code>0.2.0</code>.
    </blockquote>
</div>

<hr>
<br>

## Scope

Passwords (Argon2id, rehash on parameter change, optional pepper), API keys (keyed fast hash, scopes, expiry), sessions (256-bit tokens stored hashed, idle and absolute expiry, rotation, revocation that reaches requests in flight through the connection gate), TOTP and recovery codes, single-use tokens for verification and reset, login budgets per account and per address prefix, and uniform responses that do not reveal whether an account exists. It does not decide permissions or store user records.

See [`dev/PLAN.md`](./dev/PLAN.md) for the full scope, what is deliberately left to other crates, and the decision log.

<hr>
<br>

## Contributing

See <a href="./dev/DIRECTIVES.md"><code>dev/DIRECTIVES.md</code></a> for engineering standards and the definition of done. Before a PR: `cargo fmt --all`, `cargo clippy --all-targets --all-features -- -D warnings`, and `cargo test --all-features` must be clean.

<br>

<div id="license">
    <h2>License</h2>
    <p>Licensed under either of</p>
    <ul>
        <li><b>Apache License, Version 2.0</b> &mdash; <a href="./LICENSE-APACHE">LICENSE-APACHE</a></li>
        <li><b>MIT License</b> &mdash; <a href="./LICENSE-MIT">LICENSE-MIT</a></li>
    </ul>
    <p>at your option.</p>
</div>

<div align="center">
  <h2></h2>
  <sup>COPYRIGHT <small>&copy;</small> 2026 <strong>James Gober.</strong></sup>
</div>
