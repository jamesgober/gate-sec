<h1 align="center">
    <img width="99" alt="Rust logo" src="https://raw.githubusercontent.com/jamesgober/rust-collection/72baabd71f00e14aa9184efcb16fa3deddda3a0a/assets/rust-logo.svg">
    <br>
    <b>gate-sec</b>
    <br>
    <sub><sup>AUTHORIZATION</sup></sub>
</h1>

<div align="center">
    <a href="https://crates.io/crates/gate-sec"><img alt="Crates.io" src="https://img.shields.io/crates/v/gate-sec"></a>
    <a href="https://crates.io/crates/gate-sec"><img alt="Downloads" src="https://img.shields.io/crates/d/gate-sec?color=%230099ff"></a>
    <a href="https://docs.rs/gate-sec"><img alt="docs.rs" src="https://img.shields.io/docsrs/gate-sec"></a>
    <a href="https://github.com/jamesgober/gate-sec/actions"><img alt="CI" src="https://github.com/jamesgober/gate-sec/actions/workflows/ci.yml/badge.svg"></a>
    <a href="https://github.com/rust-lang/rfcs/blob/master/text/2495-min-rust-version.md"><img alt="MSRV" src="https://img.shields.io/badge/MSRV-1.85%2B-blue"></a>
</div>

<br>

<div align="left">
    <p>
        <strong>gate-sec</strong> answers the second question: may this identity do this, to this thing, right now? Every action and resource declares the permission it needs; anything undeclared is refused. Roles cover the common case, attribute rules cover ownership, tenancy, time, network, and how recently the user proved who they are.
    </p>
    <p>
        Checks are object-level, not just route-level: being allowed to read invoices is not the same as being allowed to read this invoice. Decisions on sensitive actions are audited, denials are reported to the detection layer, and an administrator can ask why a decision came out the way it did. Authentication is the sister crate, <code>gate-net</code>.
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

Deny by default, declared permissions for every action, role-based permissions with hierarchy, attribute rules (owner, tenant, time window, network, fresh authentication), object-level checks, scoped keys that can only narrow their owner's rights, policy versioning, startup checks that every route declares a permission, audit of sensitive decisions, and decision explanations for administrators. It does not authenticate or load data.

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
