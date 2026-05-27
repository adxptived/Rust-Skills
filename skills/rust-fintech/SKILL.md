---
name: rust-fintech
description: |
  Rust Fintech and financial system design guidelines. Use when:
  - User works with money, financial transactions, ledger accounts, or currency
  - Handling high-precision calculations (e.g. rust_decimal)
  - Enforcing auditability, immutability, and event sourcing in ledger designs
  - Writing code requiring checked arithmetic to prevent overflow losses
  - Building payment processing, settlement, or reconciliation systems
  - Implementing FX conversion, interest calculations, or fee computation
  - Designing idempotent payment flows or webhook verification
---



## Quick Navigation

- [references/decimals.md](references/decimals.md)
- [references/ledger.md](references/ledger.md)

# Rust Financial & FinTech Systems Guide

Best practices for building accurate, compliant, and deterministic financial systems in Rust.

## Core Rules & Constraints

### 1. Zero Floating-Point Tolerance
Never use floats (`f32`, `f64`) to represent monetary values due to rounding errors and precision loss. Instead, use `rust_decimal::Decimal`.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;

// Bad: f64 is subject to rounding inaccuracies
let price: f64 = 19.99;

// Good: Exact fixed-point precision
let price: Decimal = dec!(19.99);
```

### 2. Money Value Object
Encapsulate currency and amount into a single type-safe domain boundary. Prevent addition or arithmetic operations between mismatched currencies.

```rust
use rust_decimal::Decimal;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Currency {
    USD,
    EUR,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Money {
    amount: Decimal,
    currency: Currency,
}

impl Money {
    pub fn new(amount: Decimal, currency: Currency) -> Self {
        Self { amount, currency }
    }

    pub fn add(&self, other: &Money) -> Result<Money, &'static str> {
        if self.currency != other.currency {
            return Err("Currency mismatch");
        }
        let new_amount = self.amount.checked_add(other.amount)
            .ok_or("Arithmetic overflow")?;
        
        Ok(Money::new(new_amount, self.currency.clone()))
    }
}
```

### 3. Double-Entry Integrity
Ensure that entries balance exactly: the sum of debits must equal the sum of credits.

```rust
pub struct LedgerEntry {
    pub account_id: String,
    pub amount: Decimal, // Positive for debit, negative for credit
}

pub struct Transaction {
    pub id: uuid::Uuid,
    pub entries: Vec<LedgerEntry>,
}

impl Transaction {
    pub fn validate(&self) -> Result<(), &'static str> {
        let sum: Decimal = self.entries.iter().map(|e| e.amount).sum();
        if sum.is_zero() {
            Ok(())
        } else {
            Err("Transaction is unbalanced: total sum of entries must equal zero")
        }
    }
}
```

### 4. Immutable Event Sourcing
Ledger entries must be append-only and immutable. Rather than modifying states in-place, record discrete event entries over time.

```rust
pub enum AccountEvent {
    Deposited { amount: Decimal, timestamp: chrono::DateTime<chrono::Utc> },
    Withdrawn { amount: Decimal, timestamp: chrono::DateTime<chrono::Utc> },
}

pub struct Account {
    id: String,
    balance: Decimal,
}

impl Account {
    pub fn apply(&mut self, event: &AccountEvent) {
        match event {
            AccountEvent::Deposited { amount, .. } => self.balance += amount,
            AccountEvent::Withdrawn { amount, .. } => self.balance -= amount,
        }
    }
}
```

### 5. Safe Math
Always use checked, wrapping, or saturating arithmetic methods rather than standard operators if there's any chance of input overflow.
- `checked_add` / `checked_sub`
- `saturating_add` / `saturating_sub`

## Financial Correctness Rules

### Money Representation

Prefer integer minor units for ledgers and settlement. Use decimal types for pricing, rates, tax, and calculations where scale must be preserved.

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct AmountMinor(i64);

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Currency { Usd, Eur, Gbp }
```

### Ledger Invariants

Every posted transaction must be:

- balanced per currency;
- immutable after posting;
- idempotent at external boundaries;
- traceable to actor/request/source;
- reversible through compensating entries, not mutation.

## Idempotency Pattern

```rust
pub struct IdempotencyKey(String);

pub async fn post_payment(req: PaymentRequest, key: IdempotencyKey) -> Result<Receipt, Error> {
    // 1. reserve key in transaction
    // 2. if existing response, return it
    // 3. validate and post ledger entries
    // 4. persist response tied to key
    // 5. commit
    todo_domain_example()
}
```

## Decimal/Rounding Rules

- Never use floats for money.
- Round exactly once at the domain-approved boundary.
- Store the rounding policy with calculated results when audit matters.
- Keep FX rates with source, timestamp, precision, and effective interval.

## FX Conversion Pattern

```rust
pub struct FxRate {
    pub base: Currency,
    pub quote: Currency,
    pub rate: Decimal,
    pub source: &'static str,
    pub fetched_at: chrono::DateTime<chrono::Utc>,
}

impl FxRate {
    pub fn convert(&self, amount: Decimal) -> Result<Decimal, &'static str> {
        (amount * self.rate).round_dp(4).checked_sub(Decimal::ZERO)
            .ok_or("FX conversion overflow")
    }
}

pub struct FxProvider {
    rates: HashMap<(Currency, Currency), FxRate>,
}

impl FxProvider {
    pub fn convert(&self, from: Currency, to: Currency, amount: Decimal) -> Result<Money, FxError> {
        if from == to {
            return Ok(Money::new(amount, from));
        }

        let rate = self.rates.get(&(from, to))
            .ok_or(FxError::RateNotFound { from, to })?;

        if rate.fetched_at + chrono::Duration::minutes(5) < chrono::Utc::now() {
            return Err(FxError::StaleRate);
        }

        let converted = amount * rate.rate;
        Ok(Money::new(converted.round_dp(2), to))
    }
}
```

## Fee Calculation

```rust
pub enum FeeStructure {
    Flat { amount: Decimal },
    Percentage { rate: Decimal, min: Option<Decimal>, max: Option<Decimal> },
    Tiered { tiers: Vec<(Decimal, Decimal)> },
}

impl FeeStructure {
    pub fn calculate(&self, amount: Decimal) -> Result<Decimal, FeeError> {
        match self {
            FeeStructure::Flat { amount: fee } => Ok(*fee),
            FeeStructure::Percentage { rate, min, max } => {
                let raw = (amount * rate).round_dp(2);
                let bounded = match (min, max) {
                    (Some(lo), _) if raw < *lo => *lo,
                    (_, Some(hi)) if raw > *hi => *hi,
                    _ => raw,
                };
                Ok(bounded)
            }
            FeeStructure::Tiered { tiers } => {
                for (threshold, rate) in tiers.iter().rev() {
                    if amount >= *threshold {
                        return Ok((amount * rate).round_dp(2));
                    }
                }
                Ok(Decimal::ZERO)
            }
        }
    }
}
```

## Interest Calculation

```rust
pub fn compound_interest(
    principal: Decimal,
    annual_rate: Decimal,
    periods: u32,
    compounds_per_year: u32,
) -> Result<Decimal, &'static str> {
    let rate_per_period = annual_rate / Decimal::from(compounds_per_year);
    let total_periods = periods * compounds_per_year;
    let mut balance = principal;

    for _ in 0..total_periods {
        let interest = (balance * rate_per_period).round_dp(2);
        balance = balance.checked_add(interest)
            .ok_or("Interest calculation overflow")?;
    }

    Ok(balance - principal) // total interest earned
}
```

## Webhook Signature Verification

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

pub fn verify_webhook_signature(
    payload: &[u8],
    signature: &str,
    secret: &[u8],
) -> Result<(), WebhookError> {
    let expected = hex::decode(signature)
        .map_err(|_| WebhookError::InvalidSignature)?;

    let mut mac = HmacSha256::new_from_slice(secret)
        .map_err(|_| WebhookError::InvalidSecret)?;
    mac.update(payload);

    mac.verify_slice(&expected)
        .map_err(|_| WebhookError::SignatureMismatch)
}
```

## Testing Strategy

- Property test ledger balance invariants.
- Test duplicate idempotency keys.
- Test reversal entries sum to zero.
- Test currency mismatch errors.
- Snapshot audit records for required fields.
- Integration test database constraints.
- Test rounding at domain boundaries with known trap values.
- Test FX rate staleness detection.

## Anti-Patterns

- Updating account balances without immutable entries.
- Storing money as JSON floats.
- Retrying non-idempotent payment operations.
- Combining currencies in one integer total.
- Hiding rounding in display formatting.
- Using local time for settlement boundaries.
- Expiring idempotency keys too soon (before settlement completes).
- Trusting client-provided FX rates without validation.

## Review Prompt

For fintech Rust, verify numeric representation, ledger immutability, idempotency, audit trail completeness, database constraints, and property tests for invariants.

## Production Checklist

- Money never uses binary floating point.
- Ledger updates are append-only and transactionally balanced.
- Idempotency keys protect retries at every external boundary.
- Audit records include actor, request/source, timestamp, and correlation ID.
- Rounding policy is explicit and tested at domain boundaries.
- Reversals use compensating entries instead of mutation.
- FX rates have source, timestamp, and staleness checks.
- Webhook signatures are verified with timing-safe comparison.
- All arithmetic uses checked operations.
- Test suite covers currency mismatch, overflow, and rounding edge cases.

## References

- [rust_decimal](https://docs.rs/rust_decimal)
- [bigdecimal](https://docs.rs/bigdecimal)
- [serde](https://serde.rs/)
- [proptest](https://docs.rs/proptest)
- [hmac](https://docs.rs/hmac)
