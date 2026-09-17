# nemotron-h-56b-spark

Pre-configured **Nemotron-H-56B-Base-8K** inference server for the DGX Spark — a plain **vLLM**
StatefulSet serving the bf16 checkpoint with **load-time dynamic FP8 quantization**
(`--quantization fp8`). First **Mamba-2 hybrid** and first **dynamic-FP8** workload in the kit.

Nemotron-H is NVIDIA's hybrid Mamba-Transformer family: **Mamba-2 + MLP layers with just 10
attention layers** (118 layers total, hidden 8192). The Mamba layers give linear-time inference
and tiny KV state, so memory is dominated by weights, not KV.

> **⚠️ License**: this model ships under the
> **NVIDIA Internal Scientific Research and Development Model License** — **"for research and
> development only"** (not the kit baseline Apache-2.0). Confirm your entitlement before deploying.

> **ℹ️ Base model**: completion-only — **no chat template** (no 56B instruct variant exists).
> Call OpenAI `/v1/completions` for raw continuations; chat/UIs will show raw text.

## Why dynamic FP8?

The HF checkpoint is bf16 — **104.9Gi on disk**, which alone exceeds the ~100GB usable
unified-memory budget. No FP8/INT4 checkpoint exists anywhere on HF (verified author-agnostic
searches). Dynamic FP8 halves the in-RAM weight footprint to **~52.5Gi** at load time. Quality-safe:
the paper (arXiv 2504.03624) reports **56B-Base was itself fully pre-trained with an FP8 recipe**.
Ungated — no token needed (still shipped per kit convention).

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Model | `nvidia/Nemotron-H-56B-Base-8K` (ungated) |
| Image | `vllm/vllm-openai:latest` |
| Engine flags | `--quantization fp8` (dynamic) · `--kv-cache-dtype fp8` · prefix caching · `--mamba-cache-mode align` · `--enable-chunked-prefill` · trust-remote-code |
| Context | 8192 tokens (locked — native max) |
| Concurrency | 4 sequences (locked) |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory limit | 72Gi (container OOM, never host) — footprint ~63GB (52.5Gi FP8 weights + KV + activations + runtime) |
| util ceiling | 0.55 → ~70GB (below the 72Gi cgroup, ~7GB margin) |
| Storage | 120Gi default — bf16 cache is 104.9Gi on disk (quant happens in-RAM at load) |

Ingress: `/plugin/<name>` prefix, routed through an in-pod **nginx sidecar** that strips the prefix
before proxying to vLLM (`:8000`), 600s timeouts.

## Fields

- `gpu_memory_utilization` (default `0.55` — ceiling sized to the ~63GB footprint).
- `served_model_name` (default `model`) — the model string clients send in API requests
  (the OpenAI-compatible `model` field). Standardized across all kit workloads.
- **Resources**: `cpu` and `memory` request = limit (Guaranteed QoS). Defaults 2 CPU / 72Gi.
  Guaranteed reserves the full request — 72Gi + 48Gi and 72Gi + 88Gi exceed 117.5Gi, so the node
  fits one large model at a time (second pod stays Pending until the first is deleted).
- `hf_token` (optional) — ungated model; covers private repos / future gating.
- `storage_class` (required) / `storage_size` (default 120Gi) — model cache volume. `local-path` recommended.

## Prerequisites

- `nvidia-gpu-operator` plugin from the official catalog, driver ≥ 580.
- **Slow first boot**: download 104.9Gi into the model volume, then FP8-quantize 56B params at
  load. The startup probe allows **50 minutes** (300 × 10s); restarts resume from the cached
  download.
- **Hardware validation pending** — this is an **assembled** config (vLLM Nemotron-H support
  ≥ 0.21 + the model's FP8-pretrained claim), not an NVIDIA-published Spark playbook. First launch
  ladder: node stays Ready, `2/2` pods, `/v1/models` answers `id: "model"`, peak container memory
  < 72Gi.

## Known failure signatures

- **exit 2 fast** = flag removed on `:latest` drift (first suspect: `--mamba-cache-mode` /
  `--enable-chunked-prefill` / `--quantization`). **Do not drop `--quantization fp8` to
  recover** — bf16 (~105Gi) does not fit; repackage with the updated image instead.
- CrashLoopBackOff 401 = `HF_TOKEN` wiring (not gated today, but keep the secret key named `HF_TOKEN`).