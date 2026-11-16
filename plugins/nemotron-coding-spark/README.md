# nemotron-coding-spark

Pre-configured **NVIDIA Nemotron 3.5 Lightning 30B-A3B** (NVFP4) inference server for the DGX Spark —
a plain **vLLM** StatefulSet running vLLM **v0.27.1** with **DSpark speculative decoding**, the exact
configuration NVIDIA published for DGX Spark with a 512K context window for coding.

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
| Context | **524288 tokens (512K — 4× original 131072; KV pool identical to original, footprint ~39Gi unchanged; gpu_memory_utilization: 0.40)** |
| Concurrency | **1 sequence** (KV pool 524288 × 1 = 131072 × 4; sufficient for single long coding completions) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 48Gi (container OOM, never host) — footprint ~39GB (weights + KV + overhead) |

Ingress: `/{{ .Release.Namespace }}/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`).

Larger Context for Coding: This plugin provides 512K token context windows for code completions,
entire codebases, and multi-file refactoring. The KV pool (524288 × 1 = 524288) is identical to
the original 131072 × 4, so memory footprint and gpu_memory_utilization (0.40) are unchanged.
Only 1 concurrent sequence — sufficient for single long coding completions (one at a time).

Config source: [vLLM blog — Nemotron 3.5 Lightning Day-0](https://vllm.ai/blog/2026-08-10-nemotron-3-5-lightning-vllm),
"Local Deployment on DGX Spark" section. **Hardware validation pending** — no GB10 in CI.

## Fields

- `gpu_memory_utilization` (default `0.40` — ceiling sized to the ~39Gi footprint, unchanged from original; enables 512K ctx with seq=1).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.

- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 48Gi.
  Guaranteed reserves the full request — 48Gi + 48Gi fits; two large models (88Gi+) cannot co-schedule on 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `use_hami` (boolean, default `true`) — `true` requests the GPU through the device plugin (`nvidia.com/gpu: "1"`); on a HAMI cluster HAMI then caps and accounts the pod. `false` **bypasses HAMI**: the pod carries the label `hami.io/webhook: ignore`, requests no GPU resource, and gets the GPU from `runtimeClassName: nvidia` with `NVIDIA_VISIBLE_DEVICES=all`. vLLM then sees the whole device, so `gpu_memory_utilization` `0.40` works as on a cluster without HAMI. Trade-off: nothing accounts the usage — HAMI and the scheduler consider the GPU free, so avoiding oversubscription with other GPU workloads is up to you. `gpu_cores` / `gpu_mem` are ignored when `false`.
- `gpu_cores` / `gpu_mem` (optional, **HAMI clusters only**, default empty) — render `nvidia.com/gpucores` (percent of GPU compute, 1–100) and `nvidia.com/gpumem` (MiB) on the vLLM container. Empty = not rendered, so clusters without HAMI still schedule. On a HAMI cluster, set `gpu_cores` below 100 to leave compute for other GPU workloads on the node (an unset value can be allocated all 100). vLLM sees `gpu_mem` (or HAMI's `defaultMemory` when unset) as the whole device, so `gpu_memory_utilization` is a fraction of that amount: the model plus overhead needs about 25 GiB — e.g. `gpu_mem` `32768` with `gpu_memory_utilization` `0.95`. Failure signature when the fraction is too small: `Available KV cache memory: -12.2 GiB` / `No available memory for the cache blocks`.
- `hf_token` (optional) — NVIDIA models require license acceptance; covers future gating + private repos.
- `storage_class` (required) / `storage_size` (default 100Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog (device plugin + nvidia runtime class), driver ≥ 580.
- First launch downloads the NVFP4 + DSpark checkpoints (~18GB) into the model volume.
