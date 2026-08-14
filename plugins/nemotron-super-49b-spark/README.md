# nemotron-super-49b-spark

Pre-configured **Llama-3.3-Nemotron-Super-49B-v1.5** (FP8) inference server for the DGX Spark — a
plain **vLLM** StatefulSet. Dense 49B model (NVIDIA's text-to-knowledge-graph playbook): stronger
general quality than the 30B MoE models at higher memory cost.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/Llama-3_3-Nemotron-Super-49B-v1_5-FP8` (repo id uses underscores) |
| Image | `vllm/vllm-openai:latest` |
| Engine flags | KV cache fp8 · prefix caching |
| Context | 32768 tokens (locked) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 88Gi (container OOM, never host) — footprint ~79GB (weights + KV + overhead) |

Ingress: `/plugin/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

> **Confidence note**: this configuration is **assembled** from standard vLLM flags — NVIDIA has not
> published a Spark-specific serving command for this model. The FP8 checkpoint ID was corrected to
> the HF repo name after a launch surfaced a 404 on the dotted NIM-style ID.

## Fields

- `gpu_memory_utilization` (default `0.65` — ceiling sized to the ~79GB footprint).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 88Gi.
  Guaranteed reserves the full request — 88Gi + 48Gi fits; two large models cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — NVIDIA models require license acceptance; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog, driver ≥ 580.
- First launch downloads the FP8 checkpoint (~50GB) into the model volume — the 88Gi limit contains
  the download transient.
- **Hardware validation pending** — the FP8 load at 0.75 is the single largest memory consumer in the kit.
