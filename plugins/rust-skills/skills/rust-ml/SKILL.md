---
name: rust-ml
description: Use when building machine learning or AI inference in Rust. Covers inference, model loading, tensor operations, GPU and CUDA acceleration, batch processing, feature extraction, embeddings, tokenizer, hugging face integration, deep learning, ONNX with tract, model singletons with OnceLock, candle, tch-rs, ndarray tensors, and data pipelines with polars.
---

# Machine Learning Development

## Domain Constraints

| Domain Rule | Design Constraint | Rust Implication |
|-------------|-------------------|------------------|
| Large data | Efficient memory | Zero-copy, streaming |
| GPU acceleration | CUDA/Metal support | candle, tch-rs |
| Model portability | Standard formats | ONNX |
| Batch processing | Throughput over latency | Batched inference |
| Numerical precision | Float handling | ndarray, careful f32/f64 |
| Reproducibility | Deterministic | Seeded random, versioning |

## Critical Rules

- **Avoid copying large tensors** — memory bandwidth is the bottleneck. Use references, views, in-place ops.
- **Batch operations for GPU efficiency** — GPU has overhead per kernel launch, batch to amortize.
- **Use standard model formats** — train in Python, deploy in Rust via ONNX.

## Use Case to Framework

| Use Case | Recommended | Why |
|----------|-------------|-----|
| Inference only | tract (ONNX) | Lightweight, portable |
| Training + inference | candle, burn | Pure Rust, GPU |
| PyTorch models | tch-rs | Direct bindings |
| Data pipelines | polars | Fast, lazy eval |

## Key Crates

| Purpose | Crate |
|---------|-------|
| Tensors | ndarray |
| ONNX inference | tract |
| ML framework | candle, burn |
| PyTorch bindings | tch-rs |
| Data processing | polars |
| Embeddings | fastembed |

## Inference Server Pattern

```rust
use std::sync::OnceLock;
use tract_onnx::prelude::*;

static MODEL: OnceLock<SimplePlan<TypedFact, Box<dyn TypedOp>, Graph<TypedFact, Box<dyn TypedOp>>>> = OnceLock::new();

fn get_model() -> &'static SimplePlan<...> {
    MODEL.get_or_init(|| {
        tract_onnx::onnx()
            .model_for_path("model.onnx")
            .unwrap()
            .into_optimized()
            .unwrap()
            .into_runnable()
            .unwrap()
    })
}

async fn predict(input: Vec<f32>) -> anyhow::Result<Vec<f32>> {
    let model = get_model();
    let input = tract_ndarray::arr1(&input).into_shape((1, input.len()))?;
    let result = model.run(tvec!(input.into()))?;
    Ok(result[0].to_array_view::<f32>()?.iter().copied().collect())
}
```

## Batched Inference Pattern

```rust
async fn batch_predict(inputs: Vec<Vec<f32>>, batch_size: usize) -> Vec<Vec<f32>> {
    let mut results = Vec::with_capacity(inputs.len());

    for batch in inputs.chunks(batch_size) {
        let batch_tensor = stack_inputs(batch);
        let batch_output = model.run(batch_tensor).await;
        results.extend(unstack_outputs(batch_output));
    }

    results
}
```

## Common Mistakes

| Mistake | Domain Violation | Fix |
|---------|-----------------|-----|
| Clone tensors | Memory waste | Use views |
| Single inference | GPU underutilized | Batch processing |
| Load model per request | Slow | Singleton pattern |
| Sync data loading | GPU idle | Async pipeline |
