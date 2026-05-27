# Tensors and Numerical Computing

Rust ML stacks commonly use `ndarray`, `candle`, `burn`, or bindings to native runtimes like LibTorch.

## ndarray for CPU Arrays

Use `ndarray` for CPU-side numeric arrays and scientific computing.

```rust
use ndarray::{array, Array2, ArrayBase, Dim, OwnedRepr, IxDyn};

// 2D array creation
let a: Array2<f32> = array![[1.0, 2.0], [3.0, 4.0]];
let b = a.t().dot(&a);          // (2,2) × (2,2) matrix multiply
let mean = a.mean().unwrap();   // scalar reduction
let row_sums = a.sum_axis(ndarray::Axis(1)); // per-row sums

// Reshaping and slicing
let flat: ArrayBase<OwnedRepr<f32>, Dim<IxDyn>> = a.into_shape_clone(IxDyn(&[4])).unwrap();
let slice = a.slice(s![0..2, 0..1]); // view without copy
```

## Candle for Lightweight Inference

Candle is ideal for CPU inference and Hugging Face model ports with zero native dependency.

```rust
use candle_core::{Device, Tensor, Shape};

let device = Device::Cpu;
let input = Tensor::new(&[1.0f32, 2.0, 3.0], &device)?;
let output = input.sqr()?;           // element-wise
let mat = Tensor::rand(0.0f32, 1.0, Shape::from((3, 3)), &device)?;
let result = mat.matmul(&input.reshape((3, 1))?)?; // matrix-vector

// Device transfer
let cuda = Device::new_cuda(0)?;
let input_gpu = input.to_device(&cuda)?;
```

## Burn for Training

Burn provides backend-abstracted training and inference.

```rust
use burn::tensor::{Tensor, backend::Backend};
use burn::tensor::Distribution;

fn train_model<B: Backend>(device: &B::Device) {
    let x: Tensor<B, 2> = Tensor::random([32, 784], Distribution::Normal(0.0, 1.0), device);
    let weights: Tensor<B, 2> = Tensor::random([784, 10], Distribution::Normal(0.0, 0.02), device);
    let logits = x.matmul(weights);
    // loss.backward(), optimizer.step(), etc.
}
```

## Backend Selection Matrix

| Backend | Hardware | Compile time | Maturity |
|---------|----------|-------------|----------|
| ndarray | CPU only | Fast | Stable |
| WGPU (Burn) | GPU (cross-API) | Medium | Active dev |
| CUDA (Burn) | NVIDIA GPU | Medium | Production |
| Candle CPU | CPU | Fast | Production (inference) |
| LibTorch (tch) | CPU + GPU | Slow (needs C++) | Production |

## Data Layout Awareness

```rust
// ndarray is row-major (C order) by default
let arr = array![[1, 2, 3], [4, 5, 6]];
assert_eq!(arr.as_slice().unwrap(), &[1, 2, 3, 4, 5, 6]);

// Transpose returns a view, not a copy — watch for layout changes
let t = arr.t();                   // column-major view
let t_owned = arr.t().to_owned();  // force reshape to contiguous
```

Reshaping a non-contiguous tensor may fail or silently copy. Call `.contiguous()` or `.to_owned()` before reshape.

## Batching

Batch inference to amortize overhead, but cap latency for real-time paths.

```rust
const MAX_BATCH_WAIT_MS: u64 = 5;

async fn batch_inference(model: &Model, samples: Vec<Sample>) -> Vec<Output> {
    let batch = Batch::from_samples(samples, MAX_BATCH_WAIT_MS).await;
    model.forward(batch.tensor)
}
```

## Model Loading for Large Checkpoints

```rust
// Memory-map large weights to avoid OOM during load
use std::fs::File;
use memmap2::Mmap;

fn load_checkpoint(path: &str) -> Result<Weights, Error> {
    let file = File::open(path)?;
    let mmap = unsafe { Mmap::map(&file)? };
    // Parse directly from mmap, don't copy into a Vec first
    Weights::from_bytes(&mmap)
}
```

## Quantization

- Use f16 or int8 quantized models for inference where accuracy loss is acceptable.
- Candle supports `q4_0`, `q4_1`, `q5_0`, `q5_1`, `q8_0` quantization formats.
- Burn has `F16` backend support.
- Always benchmark accuracy on your domain before deploying quantized models.

## Common Operations Reference

| Operation | ndarray | Candle | Burn |
|-----------|---------|--------|------|
| Matmul | `.dot()` | `.matmul()` | `.matmul()` |
| Element-wise | `a + b` | `a + b` | `a + b` |
| Broadcast | `.broadcast()` | `.broadcast()` | `.expand()` |
| Reduce | `.sum_axis()` | `.sum()` | `.sum_dim()` |
| Reshape | `.into_shape()` | `.reshape()` | `.reshape()` |
| Gather | `.select()` | `.index_select()` | `.select()` |
| Concatenate | `concatenate![]` | `.cat()` | `.cat()` |
