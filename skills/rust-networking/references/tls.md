# TLS/SSL

Secure socket communication with rustls (pure Rust) and native-tls (OpenSSL).

## TLS Library Comparison

| Library | Implementation | Platform | Binary size | Audit status |
|---------|---------------|----------|-------------|--------------|
| `rustls` | Rust-only (ring/aws-lc) | Cross-platform | ~500 KB | FIPS 140-3 pending |
| `native-tls` | Bindings to OS TLS | Platform-specific | Varies | Depends on platform |
| `openssl` | OpenSSL bindings | Cross-platform | ~2 MB | Widely audited |
| `s2n-tls` | AWS C implementation | Linux | ~1 MB | AWS crypto |

Prefer `rustls` for new projects: fewer CVEs, no C dependency, smaller binary.

## Rustls Server

```rust
use rustls::pki_types::CertificateDer;
use rustls::ServerConfig;
use std::sync::Arc;
use tokio_rustls::TlsAcceptor;
use tokio::net::TcpListener;

async fn tls_server() -> Result<(), Box<dyn std::error::Error>> {
    // Load certificate chain and private key
    let cert_file = &mut std::io::BufReader::new(std::fs::File::open("cert.pem")?);
    let key_file = &mut std::io::BufReader::new(std::fs::File::open("key.pem")?);

    let certs: Vec<CertificateDer> = rustls_pemfile::certs(cert_file)
        .collect::<Result<Vec<_>, _>>()?;
    let key = rustls_pemfile::private_key(key_file, rustls::crypto::ring::default_provider())
        .expect("private key")?;

    let config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)?;

    let acceptor = TlsAcceptor::from(Arc::new(config));
    let listener = TcpListener::bind("0.0.0.0:443").await?;

    loop {
        let (stream, _) = listener.accept().await?;
        let acceptor = acceptor.clone();
        tokio::spawn(async move {
            let tls_stream = acceptor.accept(stream).await.unwrap();
            handle_tls_connection(tls_stream).await;
        });
    }
}
```

## Rustls Client

```rust
use tokio_rustls::TlsConnector;
use rustls::ClientConfig;
use rustls::pki_types::ServerName;
use std::sync::Arc;

async fn tls_client() -> Result<(), Box<dyn std::error::Error>> {
    let config = ClientConfig::builder()
        .with_webpki_verifier(rustls_platform_verifier::verifier())
        .with_no_client_auth();

    let connector = TlsConnector::from(Arc::new(config));
    let stream = tokio::net::TcpStream::connect("example.com:443").await?;

    let server_name = ServerName::try_from("example.com")?;
    let tls_stream = connector.connect(server_name, stream).await?;

    // Use tls_stream as a regular async read/write
    Ok(())
}
```

## Native-TLS (OS Backend)

```rust
use tokio_native_tls::{TlsAcceptor, TlsConnector};
use native_tls::{Identity, TlsAcceptor as NativeTlsAcceptor, TlsConnector as NativeTlsConnector};

// Server
async fn native_tls_server() -> Result<(), Box<dyn std::error::Error>> {
    let cert = std::fs::read("identity.pfx")?;
    let identity = Identity::from_pkcs12(&cert, "password")?;
    let acceptor = TlsAcceptor::from(NativeTlsAcceptor::new(identity)?);
    // ... same pattern as rustls server
    Ok(())
}

// Client
async fn native_tls_client() -> Result<(), Box<dyn std::error::Error>> {
    let connector = TlsConnector::from(NativeTlsConnector::new()?);
    let stream = tokio::net::TcpStream::connect("example.com:443").await?;
    let tls_stream = connector.connect("example.com", stream).await?;
    Ok(())
}
```

## Mutual TLS (mTLS)

```rust
use rustls::ServerConfig;
use std::sync::Arc;

fn mtls_server_config(ca_cert_path: &str) -> Result<ServerConfig, Box<dyn std::error::Error>> {
    let cert_file = &mut std::io::BufReader::new(std::fs::File::open("server.cert.pem")?);
    let key_file = &mut std::io::BufReader::new(std::fs::File::open("server.key.pem")?);
    let ca_file = &mut std::io::BufReader::new(std::fs::File::open(ca_cert_path)?);

    let certs: Vec<CertificateDer> = rustls_pemfile::certs(cert_file)
        .collect::<Result<Vec<_>, _>>()?;
    let key = rustls_pemfile::private_key(key_file, rustls::crypto::ring::default_provider())
        .expect("private key")?;
    let ca_certs: Vec<CertificateDer> = rustls_pemfile::certs(ca_file)
        .collect::<Result<Vec<_>, _>>()?;

    let mut root_store = rustls::RootCertStore::empty();
    root_store.add_parsable_certificates(ca_certs);

    let config = ServerConfig::builder()
        .with_client_cert_verifier(
            rustls::server::WebPkiClientVerifier::builder(root_store.into()).build()?
        )
        .with_single_cert(certs, key)?;

    Ok(config)
}
```

## Certificate Loading Helpers

```rust
use rustls::pki_types::{CertificateDer, PrivateKeyDer};
use std::fs::File;
use std::io::BufReader;

fn load_certs(path: &str) -> Result<Vec<CertificateDer>, Box<dyn std::error::Error>> {
    let mut reader = BufReader::new(File::open(path)?);
    let certs = rustls_pemfile::certs(&mut reader)
        .collect::<Result<Vec<_>, _>>()?;
    Ok(certs)
}

fn load_private_key(path: &str) -> Result<PrivateKeyDer<'static>, Box<dyn std::error::Error>> {
    let mut reader = BufReader::new(File::open(path)?);
    // Try PKCS8, then RSA, then EC
    for item in rustls_pemfile::read_all(&mut reader) {
        match item? {
            rustls_pemfile::Item::Pkcs1Key(key) => return Ok(key.into()),
            rustls_pemfile::Item::Pkcs8Key(key) => return Ok(key.into()),
            rustls_pemfile::Item::Sec1Key(key) => return Ok(key.into()),
            _ => continue,
        }
    }
    Err("no private key found".into())
}
```

## TLS Session Caching

```rust
use rustls::crypto::ring::ticketer::Ticketer;

fn config_with_session_cache() -> ServerConfig {
    let ticketer = Ticketer::new().unwrap();
    ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .map(|mut cfg| {
            cfg.ticketer = ticketer;
            cfg
        })
        .unwrap()
}

// Client-side: sessions are cached automatically in ClientConfig
// Reuse a single ClientConfig across connections for session resumption
```

## Platform Certificate Verification

```rust
// Use the system's root CA store instead of webpki
// macOS: Keychain, Windows: Cert Store, Linux: /etc/ssl/certs

// Crate: rustls-platform-verifier
use rustls_platform_verifier::Verifier;

let config = ClientConfig::builder()
    .with_platform_verifier(Verifier::new()) // uses OS trust store
    .with_no_client_auth();
```

## Anti-Patterns

```rust
// DANGEROUS: disables certificate validation — never use in production
// client.with_dangerous_accept_invalid_certs(true)

// Bad: hardcoded certificate
const CERT_PEM: &str = "-----BEGIN CERTIFICATE-----...";

// Good: load from file or env
let cert = std::fs::read("/etc/secrets/tls.crt")?;

// Bad: using a single shared password for PKCS#12 across environments
// Bad: not rotating certificates (let them expire → prod outage)
// Bad: TLS without ALPN negotiation for HTTP/2
```
