---
name: rust-security
description: |
  Rust application security, supply-chain hardening, cryptography hygiene, secrets handling, and secure service patterns. Use when:
  - User asks about secure Rust code, threat modeling, auth, tokens, password hashing, or secrets
  - User uses crates like argon2, password-hash, ring, rustls, secrecy, zeroize, jsonwebtoken, or biscuit
  - User handles untrusted input, file paths, uploads, subprocesses, deserialization, or SSRF-prone URLs
  - User wants cargo audit, cargo deny, dependency policy, SBOMs, or supply-chain checks
  - User writes unsafe code exposed to untrusted data or FFI boundaries
  - User builds web APIs that need TLS, CORS, cookies, sessions, CSRF, rate limits, or tenant isolation
  - User implements OAuth2/OIDC, API key validation, JWT verification, or mTLS
---

# Rust Security

Practical security guidance for Rust services and libraries. Rust removes many memory-safety bugs, but it does not remove auth bugs, logic bugs, injection, weak crypto, dependency risk, or unsafe-code soundness obligations.

## Quick Navigation

- **references/crypto_secrets.md** - password hashing, tokens, TLS, key handling, zeroization
- **references/supply_chain.md** - cargo audit, cargo deny, dependency policy, SBOMs

## Golden Rules

1. Treat all external input as hostile until parsed into domain types.
2. Use reviewed crypto crates and protocols; do not design cryptography.
3. Keep secrets out of `Debug`, logs, panics, metrics, and error contexts.
4. Bound work for untrusted requests: size, depth, time, concurrency, and retries.
5. Automate dependency and license policy checks in CI.

## Threat Model First

```text
asset: session token
attacker: internet client with many accounts
entry points: login, refresh, logout, API auth middleware
controls: TLS, secure cookies, token rotation, rate limits, audit logs
failure modes: replay, theft from logs, weak signing key, missing tenant check
```

A short threat model beats scattered hardening. Identify assets, attackers, boundaries, and abuse cases before choosing crates.

## Secrets Handling

```rust
use secrecy::{ExposeSecret, SecretString};

pub struct Config {
    database_url: SecretString,
}

async fn connect(config: &Config) -> Result<(), sqlx::Error> {
    sqlx::PgPool::connect(config.database_url.expose_secret()).await?;
    Ok(())
}
```

Use secret wrapper types to avoid accidental `Debug` output. Avoid attaching raw secrets to errors with `.context()`.

## Memory Zeroization

```rust
use zeroize::{Zeroize, ZeroizeOnDrop, Zeroizing};

// Automatic zeroing on drop
let mut key = Zeroizing::new([0u8; 32]);
key.copy_from_slice(&raw_key);

// Custom types can implement Zeroize
impl Zeroize for SessionKey {
    fn zeroize(&mut self) {
        self.inner.zeroize();
        self.created_at.zeroize();
    }
}
```

Use `zeroize` for cryptographic material. Relying on the OS to reclaim memory is not sufficient.

## Password Hashing

```rust
use argon2::{Argon2, PasswordHash, PasswordHasher, PasswordVerifier};
use password_hash::{rand_core::OsRng, SaltString};

pub fn hash_password(password: &[u8]) -> Result<String, password_hash::Error> {
    let salt = SaltString::generate(&mut OsRng);
    Ok(Argon2::default().hash_password(password, &salt)?.to_string())
}

pub fn verify_password(password: &[u8], encoded: &str) -> Result<bool, password_hash::Error> {
    let parsed = PasswordHash::new(encoded)?;
    Ok(Argon2::default().verify_password(password, &parsed).is_ok())
}
```

Use password hashing algorithms for passwords, not fast hashes. Tune cost parameters for the deployment budget.

## JWT Verification

```rust
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm, TokenData};

#[derive(Debug, serde::Deserialize)]
pub struct Claims {
    pub sub: String,
    pub exp: usize,
    pub iss: String,
    pub aud: String,
}

pub fn verify_token(token: &str, public_key: &[u8]) -> Result<Claims, jsonwebtoken::errors::Error> {
    let mut validation = Validation::new(Algorithm::RS256);
    validation.set_audience(&["my-service"]);
    validation.iss = Some("auth.my-service.com".to_string());

    let token_data: TokenData<Claims> = decode(
        token,
        &DecodingKey::from_rsa_pem(public_key)?,
        &validation,
    )?;

    Ok(token_data.claims)
}
```

Always validate `exp`, `iss`, and `aud`. Never accept `alg: "none"`. Use asymmetric keys (RS256/ES256) so services verify without holding signing keys.

## TLS Configuration

```rust
use rustls::ClientConfig;
use rustls_platform_verifier::tls_config;

pub fn tls_client() -> ClientConfig {
    // Use the OS-native certificate verifier
    tls_config()
        .with_client_auth(none)  // no client certificate
        .expect("tls config")
}
```

Pin a minimum TLS version (1.2 or 1.3) and restrict cipher suites to modern options.

## Parse, Validate, Authorize

```rust
#[derive(Debug, Clone)]
pub struct TenantId(uuid::Uuid);

pub async fn load_invoice(
    auth: &AuthContext,
    tenant_id: TenantId,
    invoice_id: InvoiceId,
) -> Result<Invoice, Error> {
    auth.require_tenant(&tenant_id)?;
    repo::find_invoice(tenant_id, invoice_id).await?
        .ok_or(Error::NotFound)
}
```

Validate shape at the boundary, then authorize with domain identifiers. Do not rely on UI filtering or hidden route parameters.

## Auth Middleware Pattern

```rust
use axum::{
    extract::{FromRequestParts, Request},
    middleware::{from_fn, Next},
    response::Response,
};

pub async fn auth_middleware(
    mut req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = req.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let claims = verify_token(token, &PUBLIC_KEY)
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    req.extensions_mut().insert(AuthUser {
        user_id: claims.sub,
        tenant_id: claims.aud,
    });

    Ok(next.run(req).await)
}
```

## Rate Limiting

```rust
use governor::{DefaultDirectRateLimiter, Quota, RateLimiter};
use nonzero_ext::nonzero;
use std::num::NonZeroU32;

// Allow 100 requests per second
let limiter = RateLimiter::direct(Quota::per_second(
    NonZeroU32::new(100).unwrap(),
));

async fn handle_request() -> Result<(), StatusCode> {
    if limiter.check().is_err() {
        return Err(StatusCode::TOO_MANY_REQUESTS);
    }
    // process request
    Ok(())
}
```

Use token-bucket or sliding-window ratelimiters. Expose rate limit headers (`X-RateLimit-Remaining`, `Retry-After`) for client cooperation.

## Input Validation with Bounds

```rust
// Use a dedicated validation crate for complex schemas
use validator::{Validate, ValidationError};

#[derive(Debug, Validate, serde::Deserialize)]
pub struct SignupRequest {
    #[validate(length(min = 3, max = 100))]
    pub username: String,

    #[validate(email)]
    pub email: String,

    #[validate(length(min = 8, max = 256))]
    pub password: String,
}
```

Validate at the boundary with explicit constraints. Combine with size limits at the transport layer.

## Untrusted Input Bounds

```rust
pub fn parse_payload(bytes: &[u8]) -> Result<Event, Error> {
    if bytes.len() > 64 * 1024 {
        return Err(Error::PayloadTooLarge);
    }

    let value: serde_json::Value = serde_json::from_slice(bytes)?;
    parse_event(value)
}
```

Bound payload sizes, recursion depth, decompression output, regex complexity, and database result sizes.

## SSRF Protection

```rust
use url::Url;

pub fn validate_outbound_url(raw: &str) -> Result<Url, Error> {
    let url = Url::parse(raw)?;

    let scheme = url.scheme();
    if scheme != "https" && scheme != "http" {
        return Err(Error::DisallowedScheme(scheme.to_string()));
    }

    let host = url.host_str().ok_or(Error::MissingHost)?;
    if is_private_ip(host) {
        return Err(Error::BlockedPrivateHost(host.to_string()));
    }

    Ok(url)
}

fn is_private_ip(host: &str) -> bool {
    // Parse and check against 10.x.x.x, 172.16-31.x.x, 192.168.x.x, ::1, 127.x.x.x
    // Consider using a crate like `is-private` or `ipnet`
    false // implementation omitted for brevity
}
```

Prevent SSRF by filtering private IPs, loopback, and metadata endpoints before making outbound HTTP calls.

## Path Safety

```rust
use camino::{Utf8Path, Utf8PathBuf};

pub fn resolve_upload(root: &Utf8Path, name: &str) -> Result<Utf8PathBuf, Error> {
    if name.contains('/') || name.contains('\\') || name == "." || name == ".." {
        return Err(Error::InvalidPath);
    }
    Ok(root.join(name))
}
```

Do not join user-provided paths directly. Prefer opaque IDs and server-generated filenames.

## Dependency Supply Chain

```bash
# CI steps for every Rust project
cargo audit                  # known vulnerabilities
cargo deny check             # deny list / license policy / advisory DB
cargo sbom --output cyclonedx --format json  # SBOM generation
```

```toml
# .cargo/deny.toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"

[licenses]
allow = ["MIT", "Apache-2.0", "ISC", "BSD-3-Clause"]
deny = ["GPL-3.0", "AGPL-3.0"]

[bans]
multiple-versions = "deny"
wildcards = "deny"
deny = ["ansi_term"]
```

## Anti-Patterns

```rust
// Bad: fast hash for password storage.
let digest = sha2::Sha256::digest(password);

// Good: memory-hard password hashing.
let hash = Argon2::default().hash_password(password, &salt)?;
```

```rust
// Bad: secret can leak through logs.
#[derive(Debug)]
struct Login { password: String }

// Good: wrapper prevents accidental Debug exposure.
struct Login { password: secrecy::SecretString }
```

```rust
// Bad: unchecked token deserialization, no audience/issuer validation.
let claims: Claims = decode_header(token).into();

// Good: full validation with audience, issuer, expiration.
let token = decode::<Claims>(token, &key, &Validation::new(Algorithm::RS256))?;
```

```rust
// Bad: comparing secrets with non-constant-time comparison.
if input_token == stored_token { /* ... */ }

// Good: constant-time comparison prevents timing attacks.
use subtle::ConstantTimeEq;
let result = input_token.as_bytes().ct_eq(stored_token.as_bytes());
```

## Security Checklist

- Threat model names assets, attackers, trust boundaries, and abuse cases.
- Secrets use wrapper types and are excluded from logs/errors/debug output.
- Passwords use Argon2id/bcrypt/scrypt with per-password salts.
- Tokens have expiration, audience/issuer checks, key rotation plan, and replay controls.
- Untrusted input has explicit size/time/depth bounds.
- Authorization checks use server-side domain identifiers.
- CI runs dependency, license, and advisory checks.
- TLS minimum version 1.2, cipher suites restricted to modern set.
- Rate limiting is applied to authentication and resource-intensive endpoints.
- SSRF protection is in place for all outbound HTTP clients.
- Cryptographic keys use zeroize for memory cleanup.

## References

- [RustSec Advisory Database](https://rustsec.org/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [cargo-audit](https://github.com/rustsec/rustsec/tree/main/cargo-audit)
- [cargo-deny](https://github.com/EmbarkStudios/cargo-deny)
- [secrecy crate](https://docs.rs/secrecy)
- [zeroize crate](https://docs.rs/zeroize)
