# DataFrames with Polars

Use Polars for columnar data processing, lazy query optimization, and fast CSV/Parquet IO.

## Lazy Queries

Prefer lazy frames for non-trivial pipelines — the optimizer pushes filters and projections to reduce IO.

```rust
use polars::prelude::*;

let df = LazyCsvReader::new("events.csv")
    .has_header(true)
    .finish()?
    .filter(col("event_type").eq(lit("purchase")))
    .group_by([col("country")])
    .agg([col("amount").sum().alias("revenue")])
    .collect()?;
```

## Eager vs Lazy

| | Eager | Lazy |
|---|-------|------|
| API | `DataFrame` methods | `LazyFrame` → `.collect()` |
| Optimization | Manual | Automatic (predicate pushdown, projection pushdown) |
| When to use | Interactive exploration, small data | Production pipelines, large data |
| Chaining | Immediate execution | Query plan built, then optimized |

## Schema Control

Define schemas for production ingestion instead of relying only on inference.

```rust
use polars::prelude::*;
use std::fs::File;

let schema = Schema::from_iter(vec![
    Field::new("timestamp", DataType::Datetime(TimeUnit::Microseconds, None)),
    Field::new("user_id", DataType::UInt64),
    Field::new("event_type", DataType::String),
    Field::new("amount", DataType::Float64),
    Field::new("country", DataType::String),
]);

let df = LazyCsvReader::new("events.csv")
    .has_header(true)
    .with_schema(Some(schema.into()))
    .finish()?
    .collect()?;
```

## Null Handling

```rust
// Fill nulls with a specific value
df.lazy()
    .with_column(col("amount").fill_null(lit(0.0)))
    .collect()?;

// Fill nulls with the forward value
df.lazy()
    .with_column(col("amount").forward_fill(None))
    .collect()?;

// Drop rows where any column is null
df.drop_nulls::<String>(None);

// Drop rows where specific columns are null
df.drop_nulls(Some(vec!["user_id".to_string(), "amount".to_string()]));
```

## Joins

```rust
let joined = left.lazy()
    .join(
        right.lazy(),
        [col("user_id")],
        [col("id")],
        JoinType::Inner,
    )
    .collect()?;

// Validate row counts after join
assert_eq!(joined.height(), left.height(), "join should not fan out");
```

## Expressions Over Row Loops

```rust
// Bad: row iteration
let mut results = vec![];
for row in df.iter_rows() {
    let value: f64 = row[2]; // magic index
    results.push(value * 1.1);
}

// Good: columnar expression
let result = df.lazy()
    .select([(col("amount") * lit(1.1)).alias("adjusted")])
    .collect()?;
```

## IO Performance

```rust
// Parquet — fast, compressed, columnar
let df = LazyFrame::scan_parquet("data/*.parquet", ScanArgsParquet::default())?
    .collect()?;
df.write_parquet("output.parquet", Default::default())?;

// CSV — flexible but slower
let mut file = File::create("output.csv")?;
CsvWriter::new(&mut file)
    .has_header(true)
    .with_separator(b',')
    .finish(&mut df)?;
```

### Format Comparison

| Format | Read speed | Compression | Schema | Use case |
|--------|-----------|-------------|--------|----------|
| Parquet | Fast | High (columnar) | Embedded | Analytics, storage |
| CSV | Slow | None | Implicit | Interop, debugging |
| IPC/Arrow | Fastest | Low | Embedded | Zero-copy, in-memory |
| JSON | Slow | None | Flexible | API boundaries |

## Streaming for Large Datasets

```rust
// Process data in chunks without loading everything into memory
let q = LazyFrame::scan_parquet("huge_dataset/*.parquet", ScanArgsParquet::default())?
    .group_by([col("category")])
    .agg([col("value").mean()]);

// Streaming groups results without loading entire dataset
let mut stream = q.streaming().collect_stream()?;
while let Some(batch) = stream.next() {
    process_batch(batch?);
}
```

## Common Anti-Patterns

```rust
// Bad: collecting too early
let df = LazyCsvReader::new("big.csv").finish()?.collect()?; // loads everything
let filtered = df.lazy().filter(col("x").gt(lit(5))).collect()?;

// Good: filter pushed down to scan
let filtered = LazyCsvReader::new("big.csv")
    .finish()?
    .filter(col("x").gt(lit(5)))
    .collect()?;

// Bad: row-wise apply in Rust loop
// Good: use .apply() with a closure
df.lazy()
    .with_column(
        col("name")
            .apply(|s| Ok(Some(s.to_uppercase())), GetOutput::same_type())
            .alias("name_upper"),
    )
    .collect()?;
```
