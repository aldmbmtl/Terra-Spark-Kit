# nemotron-nano-spark

Pre-configured **NVIDIA Nemotron 3 Nano 30B-A3B** (NVFP4, Quantization-Aware Distillation) inference
server for the DGX Spark — a plain **vLLM** StatefulSet. Config follows the official NVIDIA DGX
Spark playbook (`build.nvidia.com/spark/nemotron`) and the vLLM blog guide for Nemotron 3 Nano.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/NVIDIA-Nemotron-3-Nano-30B-A3B-NVFP4` |
| Image | `vllm/vllm-openai:latest` |
| Engine flags | FLASHINFER (`VLLM_ATTENTION_BACKEND`) · KV cache fp8 · prefix caching · auto-tool-choice (qwen3_coder) · deepseek_r1 reasoning parser |
| Context | 131072 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 48Gi (container OOM, never host) — footprint ~38GB (weights + KV + overhead) |

Ingress: `/plugin/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

## Fields

- `gpu_memory_utilization` (default `0.40` — ceiling sized to the ~38GB footprint).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 48Gi.
  Guaranteed reserves the full request — 48Gi + 48Gi fits; two large models (88Gi+) cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — NVIDIA models require license acceptance; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 50Gi) — model cache volume. `local-path` recommended
  (direct NVMe, no longhorn instance-manager memory overhead).

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog (device plugin + nvidia runtime class), driver ≥ 580.
- First launch downloads the NVFP4 checkpoint (~17GB) into the model volume — transient disk+memory pressure; the 48Gi limit contains it.
- **Hardware validation pending** — config transcribed from published NVIDIA/vLLM commands; no GB10 in CI.
