# DeepSeek-V4-Flash on a Dual DGX Spark Cluster

Reproducible recipe for serving the **official `deepseek-ai/DeepSeek-V4-Flash`** across **two NVIDIA DGX Spark (GB10)** nodes with tensor parallelism, MTP speculative decoding, fp8 KV cache, and a 200K context window.

## Hardware
- 2x NVIDIA DGX Spark (GB10), **1 GPU per node**
- Nodes bridged over a **QSFP56 200G** link (link-local `169.254.x` addresses)
- ~149 GB model sharded across the two nodes (TP=2)

## Stack
- vLLM fork: `jasl/vllm` @ `dda4668b59567416f86956cfe7bbc1eab371a61e`
- Container/orchestration: `eugr/spark-vllm-docker` (PR #219 lineage)
- DeepSeek V4 tokenizer / tool-call / reasoning parsers, `deepseek_mtp` speculative decoding (`num_speculative_tokens: 2`), fp8 KV cache, expert parallel, `--load-format safetensors`

## CRITICAL: must run no-Ray multi-node
Each Spark has only **one GPU**, so this is a 1-GPU-per-node, 2-node deployment. It **must** be launched in **no-Ray** mode (multi-node vLLM without Ray, `--distributed-executor-backend mp`, rank-0 head + rank-1 worker). If launched via the default Ray path, vLLM mis-detects topology and fails with:

`AssertionError: local_world_size (2) must be less than or equal to the number of visible devices (1).`

With `eugr/spark-vllm-docker`'s `run-recipe.py`, pass `--no-ray`:

```
DOTENV_CONTAINER_NAME=vllm_deepseek_v4_flash \
  ./run-recipe.sh recipes/deepseek-v4-flash.yaml --no-ray
```

`.env` (see `.env.example`) supplies the cluster node IPs over the 200G link.

## Performance (measured)
- ~41 tokens/sec decode, single-stream
- 200K context, MTP=2, fp8 KV
- Cold start ≈ 6 min (149 GB load across 2 nodes); subsequent loads are faster with the model warm in page cache

## Files
- `recipes/deepseek-v4-flash.yaml` — the recipe (vLLM args, env, build refs)
- `.env.example` — dual-node cluster config template

## Credits
Built on [`eugr/spark-vllm-docker`](https://github.com/eugr/spark-vllm-docker) and the [`jasl/vllm`](https://github.com/jasl/vllm) fork. Recipe verified on a production dual-Spark deployment.
