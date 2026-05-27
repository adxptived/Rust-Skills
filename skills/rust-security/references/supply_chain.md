# Supply Chain Security

Rust dependency risk comes from advisories, abandoned crates, surprising features, native build scripts, duplicate versions, licenses, and transitive dependency sprawl.

## Baseline CI Checks

```bash
cargo install cargo-audit cargo-deny cargo-outdated
cargo audit
cargo deny check
cargo tree -d
```

Use CI to fail on known vulnerabilities and denied licenses. Review duplicate dependency versions before optimizing compile time or binary size.

## cargo-deny

```toml
# deny.toml
[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
yanked = "deny"
ignore = []

[licenses]
unlicensed = "deny"
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "ISC"]

[bans]
multiple-versions = "warn"
wildcards = "deny"
highlight = "all"
```

Keep ignores time-boxed with comments explaining the mitigation.

## Feature Hygiene

```toml
# Bad: unknown default feature surface.
reqwest = "0.12"

# Better: choose TLS/backend behavior explicitly.
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
```

Default features can pull in native TLS, compression, proxies, or runtime integrations. Audit features for network-facing and security-sensitive services.

## Build Scripts

Crates with `build.rs` execute code at build time. Treat new build scripts as a supply-chain boundary.

```bash
cargo tree --edges build
```

Review native dependencies, generated code, downloaded artifacts, and environment-variable based behavior.

## Reproducibility

```bash
cargo build --release --locked
cargo metadata --locked --format-version 1
```

Use `--locked` in CI and release builds. Commit `Cargo.lock` for binaries and services. Libraries may omit it, but CI should still test a locked resolution path when possible.

## SBOM and Inventory

For production services, keep an inventory of dependencies and versions shipped in each release. Use tools such as `cargo auditable`, CycloneDX generators, or platform-native SBOM support when required.

## Review Checklist

- `cargo audit` and `cargo deny check` run in CI.
- New dependencies justify maintenance, license, transitive size, and unsafe usage.
- Default features are reviewed and disabled when not needed.
- Build scripts and native dependencies are explicitly reviewed.
- `Cargo.lock` is committed for deployable binaries/services.
- Advisory ignores have owner, reason, and expiration.

## Anti-Patterns

- Adding crates for tiny functions without reviewing transitive dependencies.
- Ignoring yanked crates because builds still work locally.
- Using wildcard versions in production manifests.
- Allowing advisory ignores with no expiration.
- Reviewing direct dependencies while ignoring feature-enabled transitive crates.
