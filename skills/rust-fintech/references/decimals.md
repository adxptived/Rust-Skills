# Decimal Arithmetic in Rust Fintech

Financial systems must avoid binary floating point for money.

## Avoid `f32` and `f64`

```rust
let bad = 0.1_f64 + 0.2_f64; // not exactly 0.3
```

Binary floating point cannot represent most decimal fractions exactly. This causes silent rounding errors in financial calculations.

## Minor Units

Store cents, pence, satoshis, or another explicit minor unit as integers.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Money {
    amount_minor: i64,
    currency: Currency,
}

impl Money {
    pub fn from_minor(amount: i64, currency: Currency) -> Self {
        Self { amount_minor: amount, currency }
    }

    pub fn from_major(amount: f64, currency: Currency) -> Result<Self, MoneyError> {
        let scale = currency.minor_units();
        let scaled = (amount * scale as f64).round() as i64;
        Ok(Self { amount_minor: scaled, currency })
    }

    pub fn amount_minor(&self) -> i64 { self.amount_minor }
    pub fn currency(&self) -> Currency { self.currency }
}
```

## Decimal Crates

Use `rust_decimal` when scale matters and you need arbitrary precision.

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use std::str::FromStr;

// Construction
let price = dec!(12.34);
let tax_rate = Decimal::new(825, 4); // 0.0825
let from_str = Decimal::from_str("99.99")?;

// Arithmetic
let tax = price * tax_rate;
let total = price + tax;

// Rounding with explicit policy
let rounded = total.round_dp_with_strategy(
    2,
    rust_decimal::RoundingStrategy::MidpointAwayFromZero,
);

// Comparison
assert!(price > dec!(10.00));
```

Persist decimals as strings, scaled integers, or database `NUMERIC`, not JSON floats. The `serde` feature with string serialization is recommended:

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Invoice {
    #[serde(with = "rust_decimal::serde::str")]
    amount: Decimal,
}
```

## Rounding Policy

Make rounding explicit and domain-approved. Common strategies:

| Strategy | Description | Use Case |
|----------|-------------|----------|
| `MidpointAwayFromZero` | 0.5 rounds away from zero | General purpose |
| `MidpointNearestEven` (bankers) | 0.5 rounds to nearest even | Accounting, reduces bias |
| `MidpointTowardZero` | 0.5 truncates toward zero | Tax calculations |
| `MidpointNegativeInfinity` | 0.5 rounds toward negative infinity | Floor in floor plans |

```rust
pub fn calculate_tax(amount: Decimal, rate: Decimal) -> Decimal {
    let tax = amount * rate;
    tax.round_dp_with_strategy(2, rust_decimal::RoundingStrategy::MidpointTowardZero)
}
```

## Arithmetic Validation

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;

pub struct SafeCalculator;

impl SafeCalculator {
    /// Add two decimal amounts, returning an error on overflow.
    pub fn checked_add(a: Decimal, b: Decimal) -> Result<Decimal, ArithmeticError> {
        a.checked_add(b).ok_or(ArithmeticError::Overflow)
    }

    /// Multiply and round in one step.
    pub fn mul_round(a: Decimal, b: Decimal, precision: u32) -> Decimal {
        (a * b).round_dp(precision)
    }

    /// Divide with explicit scale.
    pub fn div_scale(a: Decimal, b: Decimal, scale: u32) -> Result<Decimal, ArithmeticError> {
        if b.is_zero() {
            return Err(ArithmeticError::DivisionByZero);
        }
        a.checked_div(b).ok_or(ArithmeticError::DivisionByZero)
    }
}
```

## Currency Safety

Never add different currencies without conversion.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Currency {
    USD,
    EUR,
    GBP,
    JPY,
}

impl Currency {
    pub fn minor_units(&self) -> u32 {
        match self {
            Currency::JPY => 1,
            _ => 100, // most currencies have 2 decimal places
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Money {
    amount_minor: i64,
    currency: Currency,
}

impl Money {
    pub fn checked_add(self, rhs: Money) -> Result<Money, MoneyError> {
        if self.currency != rhs.currency {
            return Err(MoneyError::CurrencyMismatch {
                left: self.currency,
                right: rhs.currency,
            });
        }
        Ok(Money {
            amount_minor: self.amount_minor
                .checked_add(rhs.amount_minor)
                .ok_or(MoneyError::Overflow)?,
            currency: self.currency,
        })
    }
}
```

## Audit Requirements

- Record original input, normalized amount, currency, rounding mode, and rate source for conversions.
- Log all financial calculations with enough context to reproduce the computation.
- Store the rounding strategy used alongside the computed amount.
- Persist conversion rate and its source (central bank, market, fixed) for every cross-currency operation.

## Libraries

| Crate | Precision | Features |
|-------|-----------|----------|
| `rust_decimal` | 28 decimal digits | Rounding strategies, serde, math ops |
| `bigdecimal` | Arbitrary precision | Mathematical operations, serde |
| `fixed` | Fixed-point (compile-time) | Type-safe precision, no alloc |

## Best Practices

- Use `i64` minor units for simple cases (up to 9 quadrillion minor units).
- Use `rust_decimal` when division and intermediate precision matters.
- Always store currency alongside the amount.
- Never use `f32`/`f64` for financial amounts — even for display.
- Document the rounding policy for every financial operation.
- Test monetary calculations with known rounding traps (e.g., $0.01 / 3, repeated rounding).
