---
name: rust-ml
description: |
  Machine Learning, Data Science, and Tensor Computation in Rust. Use when:
  - Performing data analysis or data Wrangling with Polars dataframes
  - Doing numerical computation or matrix math using ndarray
  - Implementing or running Deep Learning model inference using Burn or Candle
  - Training models or parsing dataset features in Rust
  - Building feature pipelines, batch inference services, or online prediction endpoints
  - Comparing ML frameworks for deployment: ONNX, TFLite, or native Rust crates
  - Evaluating model performance, drift detection, or A/B experimentation in Rust services
---

## Quick Navigation

- [references/dataframes.md](references/dataframes.md)
- [references/tensors.md](references/tensors.md)

# Rust Machine Learning & Data Science Guide

Strategies for high-performance numerical computing and model inference in Rust.

## Core Libraries & Rules

### 1. High-Performance DataFrames: Polars
Avoid writing manual parsing and filtering loops. Use `polars` for lightning-fast, multi-threaded columnar data operations.

```rust
use polars::prelude::*;

fn analyze_data() -> Result<DataFrame, PolarsError> {
    // Read CSV, select columns, filter rows, and execute lazily
    LazyCsvReader::new("data.csv")
        .has_header(true)
        .finish()?
        .select([col("age"), col("salary"), col("department")])
        .filter(col("age").gt(30))
        .groupby([col("department")])
        .agg([col("salary").mean().alias("avg_salary")])
        .collect()
}
```

### 2. Matrix & Tensor Math: ndarray
Use `ndarray` for multidimensional arrays, vector arithmetic, and linear algebra.

```rust
use ndarray::{array, Array2, Array, IxDyn};

// Create a 2D matrix
let matrix_a: Array2<f64> = array![
    [1.0, 2.0],
    [3.0, 4.0]
];

let matrix_b: Array2<f64> = array![
    [2.0, 0.0],
    [1.0, 2.0]
];

// Perform matrix multiplication
let result = matrix_a.dot(&matrix_b);

// Dynamic-dimensional tensor for variable shapes
let dynamic: Array<f64, IxDyn> = Array::from_shape_vec(
    IxDyn(&[2, 3, 4]),
    (0..24).map(|x| x as f64).collect(),
)?;
```

### 3. Deep Learning Frameworks: Burn vs Candle
- **`candle`**: HuggingFace's minimalist tensor library, optimized for lightweight CPU/GPU model inference (LLMs, Whisper, etc.).
- **`burn`**: Flexible, production-grade deep learning framework supporting both training and inference with type-safe backend abstractions (WGPU, LibTorch, WebGPU).
- **`ort`**: ONNX Runtime bindings for models exported from PyTorch, TensorFlow, or scikit-learn.
- **`tract`**: Inference engine for ONNX and TensorFlow models, optimized for embedded/low-latency.

#### Framework Selection Guide

| Framework | Best For | GPU Support | Training | Model Format |
|-----------|----------|-------------|----------|--------------|
| Candle   | Inference, LLMs | CUDA | No | SafeTensors, GGML |
| Burn     | Full training & inference | WGPU, LibTorch, Vulkan | Yes | Native format, ONNX |
| ORT      | ONNX model deployment | CUDA, DirectML | No | ONNX |
| Tract    | Embedded, low-latency | CPU only | No | ONNX, TF |
| Tch-rs   | PyTorch bindings | CUDA | Yes | PyTorch checkpoint |

#### Example: Inference using Candle
```rust
use candle_core::{Device, Tensor};

fn run_inference() -> candle_core::Result<()> {
    let device = Device::Cpu;
    // Create inputs from slices
    let x = Tensor::new(&[1.0f32, 2.0f32, 3.0f32, 4.0f32], &device)?;
    
    // Perform operations
    let y = x.mul(2.0)?;
    println!("Output: {:?}", y.to_vec1::<f32>()?);
    Ok(())
}
```

#### Example: ONNX Inference with ORT
```rust
use ort::{Environment, Session, Value, inputs};

fn run_onnx(model_path: &str) -> ort::Result<()> {
    let environment = Environment::builder()
        .with_name("inference_env")
        .build()?;

    let session = Session::builder()?
        .with_model_from_file(model_path)?;

    let input_tensor = Value::from_tensor(
        ndarray::Array::from_shape_vec(
            (1, 3, 224, 224),
            vec![0.0f32; 3 * 224 * 224],
        )?
    )?;

    let outputs = session.run(inputs!["input" => input_tensor]?)?;
    Ok(())
}
```

### 4. Efficient Model Loading
Keep model weights out of memory allocations during parsing. Load model weights using Memory Maps (`memmap2`) to let the OS handle virtual memory paging efficiently.

```rust
use memmap2::Mmap;

pub struct ModelWeights {
    data: Mmap,
}

impl ModelWeights {
    pub fn from_file(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let file = std::fs::File::open(path)?;
        // SAFETY: file is not modified while mapped; data is read-only.
        let data = unsafe { Mmap::map(&file)? };
        Ok(Self { data })
    }

    pub fn get_slice(&self, offset: usize, len: usize) -> &[u8] {
        &self.data[offset..offset + len]
    }
}
```

## Feature Engineering Pipelines

```rust
use polars::prelude::*;

fn build_features(path: &str) -> Result<DataFrame, PolarsError> {
    LazyCsvReader::new(path)
        .has_header(true)
        .finish()?
        .filter(col("label").is_not_null())
        .with_column(
            col("timestamp")
                .str()
                .to_datetime(None, None, lit(""),
                    StrptimeOptions::default())
                .alias("parsed_ts")
        )
        .with_column(
            (col("feature_a") - col("feature_a").mean())
                / col("feature_a").std()
                .alias("feature_a_scaled")
        )
        .select([
            col("feature_a_scaled"),
            col("feature_b"),
            col("label"),
        ])
        .collect()
}
```

## Inference Service Pattern

```rust
use std::sync::Arc;

pub struct InferenceService {
    model: Arc<ModelWeights>,
    config: ModelConfig,
}

impl InferenceService {
    pub fn new(model_path: &str, config: ModelConfig) -> Result<Self, Error> {
        let model = ModelWeights::from_file(model_path)?;
        Ok(Self { model: Arc::new(model), config })
    }

    pub async fn predict(&self, input: &[f32]) -> Result<Vec<f32>, Error> {
        // Batch input validation
        if input.len() != self.config.input_size {
            return Err(Error::InvalidInputShape);
        }

        // Run inference (could be spawned on a dedicated thread for CPU-bound work)
        let model = self.model.clone();
        let config = self.config.clone();
        let result = tokio::task::spawn_blocking(move || {
            run_inference_sync(&model, &config, input)
        }).await??;

        Ok(result)
    }
}
```

Key points:
- Load model once at startup.
- Validate model metadata and input schema.
- Batch requests where latency budget allows.
- Bound queue size.
- Record model version in every prediction log.

## Numerical Correctness

- Track shapes at boundaries.
- Normalize inputs the same way in training and serving.
- Avoid silent dtype conversions.
- Test deterministic outputs for fixed seeds/fixtures.
- Monitor drift after deployment.
- Compare floating-point results within an epsilon, not for equality.
- Log input statistics (min, max, mean, NaN count) for early drift detection.

## Performance Rules

- Prefer columnar operations over loops.
- Reuse tensor buffers when possible.
- Use memory maps for large weights.
- Choose CPU/GPU backend based on measured workload.
- Separate preprocessing latency from model latency.
- Use `rayon` for data-parallel preprocessing.
- Profile with `cargo flamegraph` to find actual bottlenecks before optimizing.

## Anti-Patterns

```rust
// Bad: loading a model per request.
async fn handle(req: Request) -> Response {
    let model = load_model("model.bin").await?; // expensive I/O on every call
    predict(model, req).await
}

// Good: load once at startup, share via Arc.
static MODEL: OnceLock<Arc<InferenceService>> = OnceLock::new();
```

```rust
// Bad: serializing the full dataframe to JSON for every response.
let json = serde_json::to_string(&df)?;

// Good: convert only what the client needs.
let summary = df.select(&["id", "score"])?.head(Some(100));
```

- Accepting arbitrary tensor shapes from clients.
- Training/serving preprocessing skew.
- Logging raw sensitive training data.
- Using floats for IDs/categories without encoding discipline.
- Silent NaN propagation through the pipeline.

## Review Prompt

For Rust ML, check data schema, preprocessing parity, model loading lifecycle, batch/latency tradeoff, backend choice, and observability for model version and drift.

## Deep Review Heuristics

When the answer depends on library choice, first identify constraints: latency budget, data size, deployment target, auditability, and team familiarity. Then recommend the smallest tool that satisfies the constraint instead of the most powerful tool.

## Production Readiness Checklist

- Inputs are validated at boundaries.
- Errors preserve enough context for debugging.
- Expensive work is measured or bounded.
- Examples avoid hidden global state.
- Security and privacy assumptions are stated.
- Tests cover edge cases and failure paths.
- Model version is logged with every prediction.
- Input/output schemas are versioned and documented.
- Preprocessing matches training pipeline (test with saved examples).
- Batch size, queue depth, and timeout are configurable without code change.

## Teaching Guidance

When explaining this topic to a user, start with the invariant, show the safe/default approach, then mention advanced alternatives only when the user's constraints justify them.

## References

- [Polars user guide](https://docs.pola.rs/)
- [Candle](https://github.com/huggingface/candle)
- [tch-rs](https://github.com/LaurentMazare/tch-rs)
- [ndarray](https://docs.rs/ndarray)
- [ort (ONNX Runtime)](https://docs.rs/ort)
- [tract](https://github.com/sonos/tract)
- [Burn](https://burn.dev/)
