# Ledger Design in Rust

A ledger records immutable financial facts. Balances are derived, not hand-edited.

## Double-Entry Model

Every transaction has entries whose signed amounts sum to zero.

```rust
pub struct Entry {
    pub account_id: AccountId,
    pub amount_minor: i64,
    pub currency: Currency,
    pub direction: Direction,
}

pub enum Direction {
    Debit,
    Credit,
}

pub struct Transaction {
    pub id: TransactionId,
    pub entries: Vec<Entry>,
    pub idempotency_key: IdempotencyKey,
    pub description: String,
}

impl Transaction {
    pub fn validate_balanced(&self) -> Result<(), LedgerError> {
        let total: i64 = self.entries.iter()
            .map(|e| match e.direction {
                Direction::Debit => e.amount_minor,
                Direction::Credit => -e.amount_minor,
            })
            .sum();
        if total == 0 {
            Ok(())
        } else {
            Err(LedgerError::Unbalanced {
                total,
                entry_count: self.entries.len(),
            })
        }
    }
}
```

## Immutability

Do not update or delete posted transactions. Correct with reversal entries.

```rust
pub struct Reversal {
    pub original_transaction_id: TransactionId,
    pub reversal_transaction: Transaction,
}

impl Ledger {
    /// Post a reversal entry that zeroes out the original transaction.
    pub fn reverse_transaction(&self, original_id: TransactionId) -> Result<Transaction, LedgerError> {
        let original = self.get_transaction(original_id)?;
        let reversal_entries: Vec<Entry> = original.entries.iter()
            .map(|e| Entry {
                direction: e.direction.opposite(),
                ..*e
            })
            .collect();

        let reversal = Transaction {
            id: TransactionId::new(),
            entries: reversal_entries,
            idempotency_key: IdempotencyKey::new(),
            description: format!("Reversal of {}", original_id),
        };

        self.post_transaction(reversal.clone())?;
        Ok(reversal)
    }
}
```

## Idempotency

External operations must carry an idempotency key. Enforce uniqueness in the database.

```sql
CREATE TABLE ledger_transactions (
    id UUID PRIMARY KEY,
    idempotency_key UUID NOT NULL,
    description TEXT NOT NULL,
    posted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX unique_ledger_idempotency
ON ledger_transactions (idempotency_key);

CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,
    transaction_id UUID NOT NULL REFERENCES ledger_transactions(id),
    account_id UUID NOT NULL,
    amount_minor BIGINT NOT NULL,
    direction VARCHAR(4) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
    currency VARCHAR(3) NOT NULL
);
```

## Aggregation Pattern

Compute account balances from entries:

```rust
pub async fn get_balance(pool: &PgPool, account_id: AccountId) -> Result<i64, sqlx::Error> {
    sqlx::query_scalar::<_, Option<i64>>(
        r#"
        SELECT
            SUM(CASE WHEN direction = 'DEBIT' THEN amount_minor ELSE -amount_minor END) AS balance
        FROM ledger_entries
        WHERE account_id = $1
        "#,
    )
    .bind(account_id.0)
    .fetch_one(pool)
    .await
    .map(|v| v.unwrap_or(0))
}
```

For high-throughput systems, maintain a materialized balance table updated atomically within the same database transaction.

## Concurrency

Use database transactions and row-level locks for posting.

```rust
pub async fn post_transaction(
    pool: &PgPool,
    tx: Transaction,
) -> Result<(), LedgerError> {
    let mut conn = pool.begin().await?;

    // Acquire advisory lock on idempotency key to prevent duplicate posting
    sqlx::query(
        "SELECT pg_advisory_xact_lock(hashtext($1))",
    )
    .bind(tx.idempotency_key.to_string())
    .execute(&mut *conn)
    .await?;

    // Check for duplicate
    let existing = sqlx::query_scalar::<_, Option<Uuid>>(
        "SELECT id FROM ledger_transactions WHERE idempotency_key = $1",
    )
    .bind(tx.idempotency_key)
    .fetch_optional(&mut *conn)
    .await?;

    if existing.is_some() {
        return Ok(()); // already posted, idempotent
    }

    // Insert transaction + entries atomically
    sqlx::query(
        "INSERT INTO ledger_transactions (id, idempotency_key, description) VALUES ($1, $2, $3)",
    )
    .bind(tx.id)
    .bind(tx.idempotency_key)
    .bind(&tx.description)
    .execute(&mut *conn)
    .await?;

    for entry in &tx.entries {
        sqlx::query(
            "INSERT INTO ledger_entries (id, transaction_id, account_id, amount_minor, direction, currency)
             VALUES ($1, $2, $3, $4, $5, $6)",
        )
        .bind(Uuid::new_v4())
        .bind(tx.id)
        .bind(entry.account_id.0)
        .bind(entry.amount_minor)
        .bind(match entry.direction {
            Direction::Debit => "DEBIT",
            Direction::Credit => "CREDIT",
        })
        .bind(entry.currency.to_string())
        .execute(&mut *conn)
        .await?;
    }

    conn.commit().await?;
    Ok(())
}
```

## Balance Reads

For correctness, compute from entries or maintain projections updated in the same transaction.

```rust
#[derive(Debug, Clone)]
pub struct AccountBalance {
    pub account_id: AccountId,
    pub balance_minor: i64,
    pub last_updated: DateTime<Utc>,
}

/// Cached balance table updated via trigger or application-level double-write.
pub async fn get_cached_balance(pool: &PgPool, account_id: AccountId) -> Result<AccountBalance, sqlx::Error> {
    sqlx::query_as!(
        AccountBalance,
        r#"
        SELECT account_id, balance_minor, last_updated
        FROM account_balances
        WHERE account_id = $1
        "#,
        account_id.0,
    )
    .fetch_one(pool)
    .await
}
```

## Audit Trail

Store actor, request id, timestamp, source system, and normalized input for every posted transaction.

```rust
pub struct AuditRecord {
    pub transaction_id: TransactionId,
    pub actor_id: String,
    pub request_id: String,
    pub source_system: String,
    pub client_ip: Option<IpAddr>,
    pub posted_at: DateTime<Utc>,
}

impl Ledger {
    pub async fn post_with_audit(
        &self,
        tx: Transaction,
        audit: AuditRecord,
    ) -> Result<(), LedgerError> {
        let mut conn = self.pool.begin().await?;

        // Post transaction (insert + entries)
        self.post_transaction_internal(&mut conn, tx).await?;

        // Write audit trail
        sqlx::query(
            "INSERT INTO ledger_audit (transaction_id, actor_id, request_id, source_system, client_ip, posted_at)
             VALUES ($1, $2, $3, $4, $5, $6)",
        )
        .bind(audit.transaction_id.0)
        .bind(&audit.actor_id)
        .bind(&audit.request_id)
        .bind(&audit.source_system)
        .bind(audit.client_ip.map(|ip| ip.to_string()))
        .bind(audit.posted_at)
        .execute(&mut *conn)
        .await?;

        conn.commit().await?;
        Ok(())
    }
}
```

## Best Practices

- Only append to the ledger; never update or delete posted entries.
- Use idempotency keys to handle retries safely.
- Use database transactions with serially isolatable snapshot for posting.
- Compute balances from entries; cache only as a performance optimization.
- Maintain a complete audit trail with actor, source, and timestamp.
- Validate balance invariants before every post (sum to zero, non-negative balances where required).
- Store amounts as minor units (i64) with explicit currency.
- Use a separate ledger table per currency or partition by currency.
