<h1 align="center">
    <img width="90px" height="auto" src="https://raw.githubusercontent.com/jamesgober/jamesgober/main/media/icons/hexagon-3.svg" alt="Triple Hexagon">
    <br><b>CHANGELOG</b>
</h1>
<p>
  All notable changes to <code>gate-net</code> will be documented in this file. The format is based on <a href="https://keepachangelog.com/en/1.1.0/">Keep a Changelog</a>,
  and this project adheres to <a href="https://semver.org/spec/v2.0.0.html/">Semantic Versioning</a>.
</p>

---

## [Unreleased]

### Changed

- `dev/PLAN.md`: sessions are explicitly server-side (section 6.1): the client
  holds only a random token, and every request is verified against the server
  record, kept in memory and backed by the database. Context binding adds a
  keyed user-agent hash alongside the address prefix and TLS fingerprint.
  Further hardening and two experiments (rolling tokens, device-bound
  sessions) listed in 6.2. Decisions D-007 and D-008; O-2 decided.

---

## [0.1.0] - 2026-10-13

The scaffold and the plan. No functional code yet.

### Added

- Crate manifest (edition 2024, MSRV 1.85, no dependencies), `Apache-2.0 OR MIT`
  license, README, REPS, CI (Linux, macOS, Windows on stable and 1.85; audit
  and deny), `deny.toml`, `clippy.toml`, `rustfmt.toml`, `.gitattributes`.
- `dev/PLAN.md` (purpose, scope, architecture, threats, ecosystem, open
  decisions, decision log), `dev/DIRECTIVES.md`, and `dev/ROADMAP.md`.

[Unreleased]: https://github.com/jamesgober/gate-net/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/jamesgober/gate-net/releases/tag/v0.1.0
