# nemotron-spark

Pre-configured **NVIDIA Nemotron 3.5 Lightning 30B-A3B** (NVFP4) inference server for the DGX Spark —
a plain **vLLM** StatefulSet running vLLM **v0.27.1** with **DSpark speculative decoding**, the exact
configuration NVIDIA published for DGX Spark.

## Why vLLM 0.27.1 (pinned)

DSpark speculative decoding needs vLLM ≥ 0.27.1. This plugin intentionally pins the validated
version — it is the one model in the kit whose config is version-locked.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-NVFP4` |
| Draft model (DSpark) | `nvidia/...-NVFP4-DSpark` |
| Image | `vllm/vllm-openai:v0.27.1` (pinned) |
| Engine flags | moe-backend marlin · KV cache fp8 · FLASHINFER · Mamba cache tuning · prefix caching · auto-tool-choice (qwen3_coder) |
| Context | 131072 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 48Gi (container OOM, never host) — footprint ~39GB (weights + KV + overhead) |

Ingress: `/plugin/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

Config source: [vLLM blog — Nemotron 3.5 Lightning Day-0](https://vllm.ai/blog/2026-08-10-nemotron-3-5-lightning-vllm),
"Local Deployment on DGX Spark" section. **Hardware validation pending** — no GB10 in CI.

## Fields

- `gpu_memory_utilization` (default `0.40` — ceiling sized to the ~39GB footprint).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 48Gi.
  Guaranteed reserves the full request — 48Gi + 48Gi fits; two large models (88Gi+) cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — NVIDIA models require license acceptance; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog (device plugin + nvidia runtime class), driver ≥ 580.
- First launch downloads the NVFP4 + DSpark checkpoints (~18GB) into the model volume.
