# Cryptography and Secrets

Security-sensitive Rust code should compose proven primitives, minimize secret exposure, and make operational key handling explicit.

## Do Not Invent Crypto

Use high-level crates and protocols. Avoid hand-rolled encryption modes, MAC composition, password hashing, random token formats, or signature verification logic.

| Need | Default direction |
|------|-------------------|
| TLS | `rustls` through framework/client integration |
| Password hashing | `argon2`, `bcrypt`, or `scrypt` via `password-hash` traits |
| Random tokens | `rand_core::OsRng` + sufficient bytes, encoded safely |
| Secret storage in memory | `secrecy`, `zeroize` for explicit cleanup needs |
| JWT/PASETO-like tokens | Verify algorithm, issuer, audience, expiry, and key ID |

## Random Tokens

```rust
use rand_core::{OsRng, RngCore};

pub fn session_token() -> String {
    let mut bytes = [0u8; 32];
    OsRng.fill_bytes(&mut bytes);
    base64_url::encode(&bytes)
}
```

Use OS randomness. Do not use timestamps, UUIDs, counters, or non-cryptographic RNGs for bearer secrets.

## Secret Types

```rust
use secrecy::{ExposeSecret, SecretString};

#[derive(Clone)]
pub struct JwtKeys {
    signing_key: SecretString,
}

impl JwtKeys {
    pub fn signing_key(&self) -> &str {
        self.signing_key.expose_secret()
    }
}
```

Only expose secrets at the call site that needs raw bytes. Avoid deriving `Debug` for config types containing secrets.

## Zeroization

```rust
use zeroize::Zeroize;

let mut key = load_key_material()?;
use_key(&key)?;
key.zeroize();
```

Zeroization helps for long-lived processes and key material, but it is not a substitute for process isolation, least privilege, or avoiding unnecessary copies.

## Password Verification

```rust
pub fn verify_login(password: &[u8], stored_hash: &str) -> Result<(), AuthError> {
    let parsed = argon2::PasswordHash::new(stored_hash)?;
    argon2::Argon2::default()
        .verify_password(password, &parsed)
        .map_err(|_| AuthError::InvalidCredentials)
}
```

Return the same public error for wrong user and wrong password. Log authentication failures without logging the submitted credential.

## Token Validation Checklist

- Signature verified with the expected algorithm.
- `exp` and `nbf` are checked with acceptable clock skew.
- `iss` and `aud` match this service.
- `sub` maps to a server-side identity that still exists.
- Key ID (`kid`) lookup cannot force SSRF or arbitrary file reads.
- Revocation or rotation behavior is documented.

## Anti-Patterns

- Hashing passwords with SHA-2, Blake3, or MD5.
- Accepting JWT algorithm from the token without an allowlist.
- Logging full authorization headers.
- Keeping API keys in panic messages or anyhow context.
- Reusing one secret for signing, encryption, sessions, and webhooks.
