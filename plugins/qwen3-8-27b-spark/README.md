# qwen3-8-27b-spark

Pre-configured **Qwen/Qwen3.8-27B** (FP8) inference server for the DGX Spark — a plain
**vLLM** StatefulSet. Qwen3.8-27B is a hybrid Mamba+Attention model (48 Mamba + 16 attention
layers) with 256K context, optimized for the Spark's unified memory with FP8 weight
quantization.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `Qwen/Qwen3.8-27B` (Apache-2.0) |
| Image | `vllm/vllm-openai:latest` |
| Quantization | FP8 (native Blackwell tensor cores, ~29GB weights) |
| Engine flags | KV cache fp8 · prefix caching · reasoning (`qwen3` parser) · auto-tool-choice (`qwen3_coder` parser) |
| Context | 262144 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 56Gi (Guaranteed QoS, ~50Gi footprint) |
| GPU util | 0.40 (~47GB CUDA ceiling) |

Ingress: `/plugin/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.40` — ~47GB ceiling, ~32GB consumed).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.
- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 56Gi.
  Guaranteed reserves the full request — 56Gi fits with one other 56Gi workload on 117.5Gi.
- `hf_token` (optional) — currently ungated; covers private repos and future gating.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume.
  `local-path` recommended.

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog, driver ≥ 580.
- First launch downloads the checkpoint into the model volume.
- **Hardware validation pending** — config transcribed from Qwen/vLLM published guidance.

## Notes

- **FP8 tradeoff for coding**: FP8 quantization halves weight precision (BF16 → FP8), trading accuracy for significantly faster decode. For interactive coding assistance (autocomplete, quick questions, refactoring), the speed gain is worth it. For generating large volumes of production-quality code where correctness matters, BF16 precision is safer — FP8 can introduce subtle errors and weaken multi-step reasoning.
