# Migrations and Schema Evolution

Database migrations are production change management, not just local setup scripts.

## Migration Rules

- Make migrations deterministic and reviewable.
- Prefer backward-compatible expand/contract changes.
- Name constraints explicitly so application code can map errors.
- Separate long data backfills from schema locks.
- Test rollback strategy even if tooling is forward-only.

## Expand / Contract

For risky schema changes, split deployment into steps:

1. Expand: add nullable column or new table.
2. Deploy app that writes both old and new shape.
3. Backfill historical rows.
4. Deploy app that reads only new shape.
5. Contract: remove old column/table after confidence window.

This avoids forcing code and schema changes to land atomically.

## SQLx Migrations

```bash
cargo install sqlx-cli --no-default-features --features rustls,postgres
sqlx migrate add create_users
sqlx migrate run
sqlx migrate revert
```

```sql
-- migrations/20260527120000_create_users.sql
create table users (
    id uuid primary key,
    email text not null,
    created_at timestamptz not null default now(),
    constraint users_email_key unique (email)
);
```

## Application Startup

```rust
pub async fn migrate(pool: &sqlx::PgPool) -> Result<(), sqlx::migrate::MigrateError> {
    sqlx::migrate!("./migrations").run(pool).await
}
```

Running migrations in-app is convenient for small deployments. For regulated or high-scale systems, run migrations as a controlled release job.

## Backfills

```rust
loop {
    let rows = sqlx::query!(
        r#"
        update users
        set normalized_email = lower(email)
        where normalized_email is null
        limit 1000
        returning id
        "#
    )
    .fetch_all(pool)
    .await?;

    if rows.is_empty() {
        break;
    }
}
```

Keep batches small, observable, restartable, and idempotent. Avoid holding one giant transaction for a whole table rewrite.

## Test Databases

- Run migrations in CI before integration tests.
- Use per-test database/schema names when tests mutate shared state.
- Prefer real PostgreSQL/MySQL/SQLite semantics over mocks for SQL behavior.
- Seed only required fixtures; large global fixtures make tests order-dependent.

## Operational Checklist

- Migration has been tested on a production-like data volume.
- Locking behavior is understood for each DDL statement.
- Backfill is restartable and has progress metrics.
- App version N and N+1 can both run during rolling deploys.
- Constraint names are explicit and stable.
- Roll-forward plan exists if revert is unsafe.

## Anti-Patterns

- Dropping columns in the same deploy that stops reading them.
- Adding `not null` columns with expensive defaults on huge tables without a plan.
- Hiding destructive migrations in application startup.
- Using anonymous constraints and then matching localized DB error text.
- Relying on ORM auto-sync in production.
