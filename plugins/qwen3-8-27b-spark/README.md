# qwen3-6-spark

Pre-configured **Qwen/Qwen3.8-27B** (BF16) inference server for the DGX Spark — a plain
**vLLM** StatefulSet. Qwen3 is the most widely used open LLM family (33.9M+ downloads on Ollama);
the 30B-A3B MoE activates only 3B parameters per token, making it very fast on the Spark's
273 GB/s unified memory.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `Qwen/Qwen3.8-27B` (Apache-2.0) |
| Image | `vllm/vllm-openai:latest` |
| Engine flags | KV cache fp8 · prefix caching · reasoning (`deepseek_r1` parser) · auto-tool-choice (hermes parser) |
| Context | 131072 tokens (locked; model supports 262144) |
| Concurrency | 2 sequences (locked) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 72Gi (container OOM, never host) — footprint ~64GB (35GB FP8 + 21.5GB KV + overhead) |

Ingress: `/plugin/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.55` — ~70GB ceiling vs ~61GB CUDA footprint).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 72Gi.
  Guaranteed reserves the full request — 72Gi + 48Gi fits; two large models cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — currently ungated; covers private repos and future gating.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume (BF16 weights ~60GB).
  `local-path` recommended.

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog, driver ≥ 580.
- First launch downloads the BF16 checkpoint (~60GB) into the model volume.
- **Hardware validation pending** — config transcribed from Qwen/vLLM published guidance.
