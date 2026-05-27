# SQLx Patterns

SQLx is a strong default for async Rust services that want explicit SQL, typed mapping, and optional compile-time query checks.

## Checked Queries

```rust
let user = sqlx::query!(
    r#"select id, email, created_at from users where id = $1"#,
    user_id
)
.fetch_optional(pool)
.await?;
```

`query!` checks SQL against a live database or an offline metadata cache. Use `cargo sqlx prepare` in CI when builds cannot access the database.

## Dynamic Queries

Use `query_as` when SQL is dynamic or compile-time checking is impractical.

```rust
#[derive(sqlx::FromRow)]
struct UserRow {
    id: uuid::Uuid,
    email: String,
}

let rows = sqlx::query_as::<_, UserRow>(
    "select id, email from users where active = $1 order by created_at desc"
)
.bind(true)
.fetch_all(pool)
.await?;
```

Keep dynamic SQL construction constrained. Prefer `QueryBuilder` over string concatenation.

## QueryBuilder

```rust
use sqlx::{Postgres, QueryBuilder};

pub async fn find_users(pool: &sqlx::PgPool, ids: &[uuid::Uuid]) -> Result<Vec<UserRow>, sqlx::Error> {
    let mut builder = QueryBuilder::<Postgres>::new("select id, email from users where id in (");
    let mut separated = builder.separated(", ");
    for id in ids {
        separated.push_bind(id);
    }
    separated.push_unseparated(")");

    builder.build_query_as::<UserRow>().fetch_all(pool).await
}
```

Never interpolate untrusted input into SQL strings.

## Transactions

```rust
pub async fn transfer(pool: &sqlx::PgPool, from: AccountId, to: AccountId, cents: i64) -> Result<(), Error> {
    let mut tx = pool.begin().await?;

    debit(&mut tx, from, cents).await?;
    credit(&mut tx, to, cents).await?;
    insert_ledger_entry(&mut tx, from, to, cents).await?;

    tx.commit().await?;
    Ok(())
}
```

Pass `&mut Transaction<'_, Postgres>` through helpers that participate in the same atomic operation.

## Pool Sizing

Pool size must fit database limits and service concurrency.

- Too small: high acquire wait time.
- Too large: database context switching and lock contention.
- Per-instance pool sizes multiply by replica count.

```text
max_connections_per_instance <= (database_limit - admin_reserved) / app_replicas
```

## Error Handling

```rust
match err {
    sqlx::Error::RowNotFound => Error::NotFound,
    sqlx::Error::Database(db) if db.constraint() == Some("users_email_key") => Error::DuplicateEmail,
    other => Error::Database(other),
}
```

Use stable constraint names in migrations so application error mapping is reliable.

## Testing

```rust
#[sqlx::test(migrations = "./migrations")]
async fn creates_user(pool: sqlx::PgPool) -> sqlx::Result<()> {
    let id = create_user(&pool, "a@example.com").await?;
    assert!(find_user(&pool, id).await?.is_some());
    Ok(())
}
```

Use SQLx test helpers for isolated databases when available. For complex setups, create per-test schemas or containers.

## Anti-Patterns

- Building SQL with `format!` and user input.
- Holding transactions across network calls.
- Spawning more concurrent DB tasks than the pool can serve.
- Treating `RowNotFound` as an exceptional internal error when it is domain-level absence.
- Hiding `pool.begin()` inside low-level helpers that need to compose.
