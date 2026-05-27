---
name: rust-database
description: |
  Rust database development with SQLx, SeaORM, Diesel, transactions, migrations, and pooling. Use when:
  - User works with PostgreSQL, MySQL, SQLite, Redis, or database-backed Rust services
  - User uses sqlx, sea-orm, diesel, refinery, sea-orm-migration, deadpool, or bb8
  - User designs repositories, transactions, migrations, schema evolution, or connection pools
  - User asks about compile-time checked SQL, query_as, FromRow, lifetimes in row mapping, or DB errors
  - User debugs pool exhaustion, deadlocks, slow queries, N+1 queries, or transaction boundaries
  - User needs test database setup, fixtures, migrations in CI, or database integration tests
  - User needs Redis caching, pub/sub, or session storage patterns
---

# Rust Database Engineering

Practical guidance for safe, observable, and testable database-backed Rust services.

## Quick Navigation

- **references/sqlx.md** - SQLx patterns, checked queries, pools, row mapping
- **references/migrations.md** - migrations, schema evolution, test databases, CI

## Golden Rules

1. Make transaction boundaries explicit.
2. Keep pool size smaller than database capacity.
3. Prefer typed IDs and domain types over raw primitives.
4. Measure query plans before adding indexes.
5. Test against the real database engine when behavior depends on SQL semantics.
6. Never interpolate user input into raw SQL strings.

## SQLx Setup

```toml
[dependencies]
sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "uuid", "chrono", "migrate"] }
uuid = { version = "1", features = ["serde", "v7"] }
thiserror = "1"
```

```rust
use sqlx::{PgPool, postgres::PgPoolOptions};

pub async fn connect(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(20)
        .min_connections(2)
        .connect(database_url)
        .await
}
```

## Typed Repository Boundary

```rust
use sqlx::{PgPool, Postgres, Transaction};
use uuid::Uuid;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(pub Uuid);

#[derive(Debug)]
pub struct User {
    pub id: UserId,
    pub email: String,
}

pub struct UsersRepo {
    pool: PgPool,
}

impl UsersRepo {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    pub async fn find_by_email(&self, email: &str) -> Result<Option<User>, sqlx::Error> {
        let row = sqlx::query!(
            r#"select id, email from users where email = $1"#,
            email
        )
        .fetch_optional(&self.pool)
        .await?;

        Ok(row.map(|row| User {
            id: UserId(row.id),
            email: row.email,
        }))
    }

    pub async fn create_in_tx(
        tx: &mut Transaction<'_, Postgres>,
        email: &str,
    ) -> Result<UserId, sqlx::Error> {
        let id = Uuid::now_v7();
        sqlx::query!(
            r#"insert into users (id, email) values ($1, $2)"#,
            id,
            email
        )
        .execute(&mut **tx)
        .await?;
        Ok(UserId(id))
    }
}
```

Keep repositories thin: SQL mapping, transaction participation, and database-specific errors. Put business decisions in services.

## Dynamic Queries with query_as

When column lists vary at runtime, use `query_as` with a raw string:

```rust
use sqlx::FromRow;

#[derive(Debug, FromRow)]
pub struct UserSummary {
    pub id: Uuid,
    pub email: String,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

pub async fn search_users(pool: &PgPool, query: &str) -> Result<Vec<UserSummary>, sqlx::Error> {
    sqlx::query_as::<_, UserSummary>(
        r#"select id, email, created_at from users where email like $1"#
    )
    .bind(format!("%{}%", query))
    .fetch_all(pool)
    .await
}
```

Use `FromRow` derive for custom result types. Prefer compile-time `query!` when the SQL is static.

## Transaction Pattern

```rust
pub async fn register_user(pool: &PgPool, email: &str) -> Result<UserId, AppError> {
    let mut tx = pool.begin().await?;

    let id = UsersRepo::create_in_tx(&mut tx, email).await?;
    AuditRepo::insert(&mut tx, "user.registered", id.0).await?;

    tx.commit().await?;
    Ok(id)
}
```

Commit only after all domain writes succeed. Rollback is automatic when `tx` drops, but explicit control flow is easier to review.

## Error Mapping

```rust
#[derive(thiserror::Error, Debug)]
pub enum AppError {
    #[error("database unavailable")]
    Database(#[from] sqlx::Error),

    #[error("email already exists")]
    DuplicateEmail,
}

fn map_db_error(err: sqlx::Error) -> AppError {
    if let sqlx::Error::Database(db) = &err {
        if db.constraint() == Some("users_email_key") {
            return AppError::DuplicateEmail;
        }
    }
    AppError::Database(err)
}
```

Map constraint names into domain errors at the boundary. Do not leak raw database messages to users.

## Pool and Query Observability

```rust
let pool = PgPoolOptions::new()
    .max_connections(20)
    .acquire_timeout(std::time::Duration::from_secs(2))
    .connect(database_url)
    .await?;
```

Track pool wait time, query latency, error count, rows affected, and transaction duration. Pool exhaustion often means slow queries, leaked transactions, or too much concurrency.

## Pagination Pattern

```rust
pub async fn list_users(
    pool: &PgPool,
    page: i64,
    page_size: i64,
) -> Result<Vec<User>, sqlx::Error> {
    sqlx::query_as::<_, User>(
        r#"select id, email from users order by id limit $1 offset $2"#
    )
    .bind(page_size)
    .bind(page * page_size)
    .fetch_all(pool)
    .await
}
```

Use keyset (cursor) pagination for large datasets instead of offset-based pagination, which degrades on deep pages.

## Redis Caching

```rust
use redis::AsyncCommands;

pub struct CacheService {
    client: redis::Client,
}

impl CacheService {
    pub async fn get_or_compute<T, F>(
        &self,
        key: &str,
        ttl_secs: usize,
        compute: F,
    ) -> Result<T, Error>
    where
        T: serde::de::DeserializeOwned + serde::Serialize,
        F: Future<Output = Result<T, Error>>,
    {
        let mut conn = self.client.get_multiplexed_async_connection().await?;

        // Try cache first
        if let Some(val) = conn.get::<_, Option<String>>(key).await? {
            return Ok(serde_json::from_str(&val)?);
        }

        // Compute and cache
        let val = compute().await?;
        let encoded = serde_json::to_string(&val)?;
        let _: () = conn.set_ex(key, encoded, ttl_secs).await?;
        Ok(val)
    }
}
```

## Database Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use sqlx::PgPool;
    use sqlx::migrate::Migrator;

    static MIGRATOR: Migrator = sqlx::migrate!(); // from migrations/ folder

    async fn test_db() -> PgPool {
        let pool = PgPoolOptions::new()
            .max_connections(1)
            .connect("postgres://localhost/test")
            .await
            .expect("test database");

        MIGRATOR.run(&pool).await.expect("migrations");
        pool
    }

    #[sqlx::test]
    async fn test_create_user() {
        let pool = test_db().await;
        let repo = UsersRepo::new(pool);
        let id = repo.create_in_tx(&mut pool.begin().await.unwrap(), "test@example.com").await.unwrap();
        assert!(id.0.to_string().len() > 0);
    }
}
```

Use SQLx's built-in `#[sqlx::test]` for per-test database transactions that roll back automatically.

## Connection Pool Sizing

| Database | Formula | Typical Max |
|----------|---------|-------------|
| PostgreSQL | `2 * (core_count + disk_count) + 1` | 20-50 per instance |
| MySQL | CPU cores * 2 + effective disk count | 50-200 |
| SQLite | 1 writer + N readers | WAL mode: 1 writer + many readers |

Monitor pool utilization. If connections are always maxed, you need more replicas or query optimization, not a bigger pool.

## Anti-Patterns

```rust
// Bad: transaction hidden inside a helper, making multi-write atomicity impossible.
async fn create_user(pool: &PgPool, email: &str) { /* begins and commits internally */ }

// Good: functions can accept &mut Transaction when the caller needs atomic composition.
async fn create_user(tx: &mut Transaction<'_, Postgres>, email: &str) { /* participates */ }
```

```rust
// Bad: N+1 queries in a loop.
for user in users {
    load_posts(pool, user.id).await?;
}

// Good: batch by IDs and map results in memory.
load_posts_for_users(pool, &user_ids).await?;
```

```rust
// Bad: string interpolation in SQL (SQL injection risk).
sqlx::query(&format!("select * from users where id = '{}'", user_id));

// Good: parameterized query.
sqlx::query!("select * from users where id = $1", user_id);
```

```rust
// Bad: holding a transaction open while doing non-database work.
let mut tx = pool.begin().await?;
call_external_api().await?; // transaction held open during external call
do_db_work(&mut tx).await?;
tx.commit().await?;

// Good: only hold transactions for database operations.
let result = call_external_api().await?;
let mut tx = pool.begin().await?;
do_db_work(&mut tx, result).await?;
tx.commit().await?;
```

## Production Checklist

- Migrations run before application startup or as a controlled release step.
- Every multi-write business operation has an explicit transaction boundary.
- Pool size, acquire timeout, and database connection limits are documented.
- Slow queries have indexes justified by `EXPLAIN`, not guesswork.
- Constraint errors are mapped into stable domain errors.
- Integration tests run against the same database engine used in production.
- Connection strings and credentials are not hardcoded — use secrets management.
- Pool metrics are exposed (used connections, wait time, total).
- Long-running transactions are logged and investigated.

## References

- [SQLx docs](https://docs.rs/sqlx)
- [Diesel guides](https://diesel.rs/guides/)
- [SeaORM docs](https://www.sea-ql.org/SeaORM/docs/)
- [PostgreSQL EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [Redis rust docs](https://docs.rs/redis)
