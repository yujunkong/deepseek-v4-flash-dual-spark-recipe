> # ⚠️ This repository is deprecated
> **The current, maintained recipe has moved to 👉 https://github.com/tonyd2wild/deepseek-v4-flash-dgx-spark**
>
> This older dual-Spark writeup is kept for history only. Please use the new repo above for the working, up-to-date DeepSeek V4 Flash on DGX Spark recipe.

---

# DeepSeek-V4-Flash on 2x DGX Spark — ~41 tok/s / 200K context

> Reproducible recipe for serving the official `deepseek-ai/DeepSeek-V4-Flash` across two NVIDIA DGX Spark (GB10) nodes with tensor parallelism (TP=2), MTP speculative decoding, fp8 KV cache, and a 200K context window.

## TL;DR

- Serves the official `deepseek-ai/DeepSeek-V4-Flash` across **2x DGX Spark (GB10)**, TP=2, ~149 GB sharded across the two nodes.
- **~41 tokens/sec** decode (single-stream), **200K context**, MTP=2 speculative decoding, fp8 KV cache.
- **Must launch in no-Ray multi-node mode** — each Spark has only one GPU, so the default Ray path mis-detects topology and fails.
- For anyone reproducing DeepSeek V4 Flash on a dual-Spark cluster.

## Hardware

- 2x NVIDIA DGX Spark (GB10), **1 GPU per node**
- Nodes bridged over a **QSFP56 200G** link (link-local `169.254.x` addresses)
- ~149 GB model sharded across the two nodes (TP=2)

## Quick start

1. Copy `.env.example` to `.env` and set your two nodes' link-local IPs on the 200G interconnect:

   ```
   CLUSTER_NODES=169.254.73.22,169.254.134.1
   LOCAL_IP=169.254.73.22
   CONTAINER_NAME=vllm_deepseek_v4_flash
   ```

2. Launch in **no-Ray** mode with `eugr/spark-vllm-docker`:

   ```
   DOTENV_CONTAINER_NAME=vllm_deepseek_v4_flash \
     ./run-recipe.sh recipes/deepseek-v4-flash.yaml --no-ray
   ```

vLLM serves an OpenAI-compatible endpoint on port `8000` as `deepseek-v4-flash`.

## Setup (detailed)

### Weights

- Model: **`deepseek-ai/DeepSeek-V4-Flash`** (official). Pulled by `vllm serve` and loaded with `--load-format safetensors`.
- ~149 GB sharded across the two nodes (TP=2).

### Image / build

Built on [`eugr/spark-vllm-docker`](https://github.com/eugr/spark-vllm-docker) (PR #219 lineage), which builds the [`jasl/vllm`](https://github.com/jasl/vllm) fork pinned via the recipe's `build_args`:

- `--vllm-repo https://github.com/jasl/vllm.git`
- `--vllm-ref dda4668b59567416f86956cfe7bbc1eab371a61e`
- `--rebuild-vllm`

The recipe wires up the DeepSeek V4 tokenizer / tool-call / reasoning parsers, `deepseek_mtp` speculative decoding (`num_speculative_tokens: 2`), fp8 KV cache, expert parallel, and `--load-format safetensors`.

### Launch

Each Spark has only **one GPU**, so this is a 1-GPU-per-node, 2-node deployment. It **must** be launched in **no-Ray** mode (multi-node vLLM without Ray, `--distributed-executor-backend mp`, rank-0 head + rank-1 worker). Pass `--no-ray` to the recipe launcher:

```
DOTENV_CONTAINER_NAME=vllm_deepseek_v4_flash \
  ./run-recipe.sh recipes/deepseek-v4-flash.yaml --no-ray
```

`.env` (see `.env.example`) supplies the cluster node IPs over the 200G link. The recipe itself lives in `recipes/deepseek-v4-flash.yaml` (vLLM args, env, build refs).

## Benchmarks

Measured on the production dual-Spark deployment:

| Metric | Value |
| --- | --- |
| Decode throughput | ~41 tokens/sec (single-stream) |
| Context window | 200K |
| Speculative decoding | MTP, `num_speculative_tokens=2` |
| KV cache | fp8 |
| Cold start | ≈ 6 min (149 GB load across 2 nodes) |

Subsequent loads are faster with the model warm in page cache.

## Configuration

Recipe defaults (`recipes/deepseek-v4-flash.yaml`):

| Setting | Value |
| --- | --- |
| `port` | 8000 |
| `host` | 0.0.0.0 |
| `tensor_parallel` | 2 |
| `pipeline_parallel` | 1 |
| `gpu_memory_utilization` | 0.8 |
| `max_model_len` | 200000 |
| `max_num_batched_tokens` | 8192 |
| `max_num_seqs` | 2 |
| `block_size` | 256 |
| `served_model_name` | deepseek-v4-flash |

Environment variables set by the recipe:

| Variable | Value |
| --- | --- |
| `TORCH_CUDA_ARCH_LIST` | 12.1a |
| `VLLM_ALLOW_LONG_MAX_MODEL_LEN` | 1 |
| `VLLM_TRITON_MLA_SPARSE` | 1 |
| `FLASHINFER_DISABLE_VERSION_CHECK` | 1 |
| `TILELANG_CLEANUP_TEMP_FILES` | 1 |
| `DG_JIT_USE_NVRTC` | 0 |
| `DG_JIT_NVCC_COMPILER` | /usr/local/cuda/bin/nvcc |
| `DG_JIT_PRINT_COMPILER_COMMAND` | 1 |
| `NCCL_IB_DISABLE` | 0 |
| `NCCL_DEBUG` | WARN |

## Troubleshooting

**Ray path mis-detects topology and fails to launch.** Each node has a single GPU, so the default Ray executor errors with:

```
AssertionError: local_world_size (2) must be less than or equal to the number of visible devices (1).
```

Fix: run in no-Ray multi-node mode by passing `--no-ray` (uses `--distributed-executor-backend mp`, rank-0 head + rank-1 worker), as shown in [Launch](#launch).

## Credits & links

Built on [`eugr/spark-vllm-docker`](https://github.com/eugr/spark-vllm-docker) and the [`jasl/vllm`](https://github.com/jasl/vllm) fork. Recipe verified on a production dual-Spark deployment.
