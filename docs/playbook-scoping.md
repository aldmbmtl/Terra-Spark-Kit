# DGX Spark Playbooks → Terra Plugin Scoping Matrix

Mapping of the 66 playbooks in [NVIDIA/dgx-spark-playbooks](https://github.com/NVIDIA/dgx-spark-playbooks)
to the Terra plugin ecosystem (this kit). Produced from a full read of every playbook README
(commit `1fb66f0`, Aug 2026).

## Scoping criteria (the gates)

A playbook is "doable" in this kit only if it satisfies the Terra plugin rules:

1. **Workload-template shape** — an always-on container workload rendered by Kuiper from
   `scripts/chart/`. Batch jobs, one-shot scripts, interactive containers do not fit the workload
   model.
2. **Ingress prefix rule** — every ingress must route `host` with path `/plugin/<name>` (pathType
   Prefix, `<name>-ingress`) + 600s nginx timeouts. Backends that cannot serve under the prefixed
   path ship the **nginx prefix-stripping sidecar** (rewrite → root, proxy to app). Apps that
   hardcode absolute paths/origins, use WebRTC (UDP), or pin `127.0.0.1` origin checks cannot be
   made prefix-compatible.
3. **vLLM-only engine policy** — LLM serving plugins are plain vLLM StatefulSets. SGLang / Ollama /
   TRT-LLM / llama.cpp / NIM / LM Studio playbooks are **policy-skipped** (no new model coverage;
   they duplicate the vLLM archetype's OpenAI-compatible surface).
4. **Memory budget** — 117.5Gi allocatable; GPU workloads are mutually exclusive (single GPU).
   Lightweight CPU-only companions (web UIs, graph DBs) co-schedule with exactly one model.
5. **No host-level dependencies** — no systemd, sudo, host network reconfiguration, physical
   cabling, desktop GUI, USB devices, or proprietary SaaS accounts.
6. **Registry reachability** — images must be pullable with an `hf_token` at most. NGC-gated images
   (NIM, VSS) need an NGC API key the kit does not carry → disqualified or Tier-2-gated.

## Tiers

| Tier | Meaning |
|------|---------|
| **TIER-1** | Clean single-container workload; fits the plugin mold with only the standard prefix-strip sidecar. |
| **TIER-2** | Fits with adaptation: custom image/build step, websocket-aware sidecar, multi-container pod, gating, or config work. |
| **TIER-3** | Disqualified: host-level, multi-node, hardware, desktop, SaaS, batch-only, or policy-skipped. |

## Buildable now (this kit)

| Plugin | Source playbook | Tier | Notes |
|--------|-----------------|------|-------|
| `open-webui-spark` | open-webui | TIER-2 | Chat UI over existing kit vLLM plugins. CPU-only companion; `OPENAI_API_BASE_URL` points at a launched model workload (`http://<model-name>:8080/plugin/<model-name>/v1`). Frontend-only image (`:main`), no bundled Ollama (policy). Sidecar: ws/SSE unbuffered locations, buffered default with sub_filter prefix rewriting (no native subpath upstream). |
| `comfy-ui-spark` | comfy-ui | TIER-2 | Node-graph image generation. DGX-native container (`mmartial/comfyui-nvidia-docker`, arm64 + cu13.1 + torch 2.10) — anonymous Docker Hub pull; venv provisioned by the image entrypoint on first boot. GPU-exclusive. ws-aware sidecar. Native-subpath frontend (≥ 1.33) — sidecar strip + 301, no sub_filter. Models managed via the UI (no auto-download). |
| `txt2kg-spark` | txt2kg | TIER-2 | Text → knowledge graph: Next.js app + ArangoDB + external LLM (vLLM field; playbook's vLLM variant defaults Nemotron-Super-49B = this kit's `nemotron-super-49b-spark`). Multi-container pod. Qdrant vector search deferred. |

## Already covered by the kit

| Playbook | Coverage |
|----------|----------|
| vllm | All 5 model plugins (nemotron-spark, nemotron-nano-spark, qwen3-spark, qwen3-6-spark, nemotron-super-49b-spark) |
| nemotron | nemotron-spark (Lightning 3.5 + DSpark), nemotron-nano-spark (Nano QAD), nemotron-super-49b-spark (Super-49B) |
| speculative-decoding | nemotron-spark's DSpark block (v0.27.1 pinned, FlashInfer, MTP-style speculative config) |

## Full matrix

### Inference servers

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| vllm | OpenAI-compat LLM serving | vLLM | 1 | ✅ covered | Already the kit's 5 model plugins |
| sglang | OpenAI-compat LLM serving (FlashInfer) | SGLang | 1 | ❌ policy | vLLM-only engine policy; duplicates serving value |
| trt-llm | TensorRT-LLM optimized serving | TRT-LLM | 2 | ❌ policy | NGC image, hostIPC/memlock, host networking; policy-skipped |
| llama-cpp | GGUF inference server | llama.cpp | 2 | ❌ policy | sm_121a custom build; policy-skipped |
| ollama | Model runner daemon | Ollama | 2 | ❌ policy | systemd host install in playbook; policy-skipped |
| nim-llm | NVIDIA NIM microservice | NIM (TRT-LLM) | 2 | ❌ gated | NGC API key required; policy-skipped |
| lm-studio | Headless LM Studio daemon | proprietary | 3 | ❌ | Host install, proprietary, no container path |
| speculative-decoding | EAGLE-3 / draft-target on TRT-LLM | TRT-LLM | 2 | ❌ policy | TRT-LLM add-on; 2-node variant TIER-3 |
| nemotron | Nano (multimodal) + Super 120B serving | vLLM/TRT-LLM | 1 | ⏳ partial | Nano text/QAD covered; Nano-**Omni** (vllm[audio]) needs custom image — future candidate; Super-120B exceeds clean budget at util 0.9 |
| nvfp4-quantization | NVFP4 quantization job + serve | ModelOpt + vLLM | 2 | ⏳ future | Quant half is a batch Job (not workload-shaped); serve half = plain vLLM. Candidate for a Job-style plugin |
| multi-modal-inference | Flux/SDXL TensorRT demo scripts | TensorRT | 3 | ❌ | Interactive one-shot scripts, no service |
| live-vlm-webui | Webcam→VLM WebRTC UI | python + backend | 3 | ❌ | WebRTC UDP + mandatory HTTPS — prefix-hostile, interactive |

### Web UIs / creative

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| comfy-ui | Node-graph image generation | ComfyUI | 2 | ✅ built | DGX-native container (mmartial, anonymous pull); ws sidecar, native-subpath frontend (≥ 1.33); models via UI |
| open-webui | Self-hosted chat UI | Open WebUI | 2 | ✅ built | Frontend-only image; sub_filter prefix sidecar (buffered default + ws/SSE locations); connects to kit vLLM via prefixed llm_url |
| txt2kg | Text → knowledge graph | Next.js + ArangoDB + LLM | 2 | ✅ built | Multi-container; app built at first boot (pinned commit); external vLLM |
| flux-finetuning | FLUX Dreambooth LoRA train + ComfyUI serve | ComfyUI | 2 | ❌ deferred | Train half = Job; ComfyUI half duplicates comfy-ui-spark; FLUX gated |
| dgx-dashboard | OEM system dashboard | host service | 3 | ❌ | Firmware/reboot, host-only OEM |
| reachy-photo-booth | Robot photo booth (13 services) | compose | 3 | ❌ | USB robot, camera, NGC login, 120GB+ footprint |
| spark-reachy-photo-booth | duplicate of above | compose | 3 | ❌ | Same |
| rag-ai-workbench | Agentic RAG via AI Workbench | Gradio + cloud APIs | 3 | ❌ | Desktop Workbench GUI + cloud API keys |

### Agents / chat / assistants

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| hermes-agent | Self-improving terminal agent + gateways | host CLI + systemd | 3 | ❌ | Interactive sudo installer, TUI, shell execution |
| openclaw | Local-first agent + dashboard | host npm gateway | 3 | ❌ | Host installer, token-URL dashboard (origin checks) |
| openshell | Sandboxed agents (Landlock) | host systemd + k3s-in-Docker | 3 | ❌ | Host daemon, sudo, interactive wizard, token-URL |
| nemoclaw | NVIDIA agent assistant | host installer + k3s-in-Docker | 3 | ❌ | Host systemd, sudo, token-fragment UI, origin pinning |
| nemoclaw-applications | 4 agent recipes on NemoClaw | prompts only | 3 | ❌ | Content on host-level platform |
| cli-coding-agent | Claude Code / Codex on local Ollama | host + systemd | 3 | ❌ | Interactive TUIs; only Ollama half containerizes (policy-skipped) |
| multi-agent-chatbot | Supervisor + coding/RAG agents + MCP | compose (11 svc) | 2 | ❌ deferred | ~120GB footprint > 117.5Gi budget; prefix-hostile SPA; 120B→20B swap needed |
| vss | Video search & summarization | compose + NGC | 3 | ❌ | sudo cache-cleaner, nvidia-ctk, NGC-gated images, multi-service |

### Fine-tuning / training (all batch — workload-shaped? no)

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| nemo-fine-tune | NeMo AutoModel SFT/LoRA/QLoRA | NGC automodel | 2 | ⏳ future | Batch Job, no always-on surface; Job-style plugin would be a new shape |
| pytorch-fine-tune | TRL SFT/LoRA/QLoRA | NGC pytorch | 2 | ⏳ future | Single-node Job OK; Swarm multi-node host-level |
| unsloth | Unsloth LoRA/QLoRA | NGC pytorch | 2 | ⏳ future | Batch Job |
| llama-factory | LLaMA Factory CLI fine-tuning | host venv | 2 | ⏳ future | Trivial to containerize; batch Job |
| flux-finetuning | (see creative) | — | 2 | ❌ deferred | Train/inference exclusive, gated |
| nvfp4-pretraining | NVFP4 pretraining (GB300) | NeMo | 3 | ❌ | Station-only, 208GB VRAM |
| station-gr00t | Isaac GR00T VLA fine-tuning | native venv | 3 | ❌ | Station-only, native env, training-only |

### Data science / notebooks (interactive-only)

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| cuda-x-data-science | cuDF/cuML notebooks | RAPIDS + Jupyter | 2 | ⏳ future | Notebook-interactive; containerizable (RAPIDS image), needs base_url |
| single-cell | GPU scRNA-seq in JupyterLab | RAPIDS + JupyterLab | 2 | ⏳ future | ≥40GB UMA, zero persistence by design, interactive |
| portfolio-optimization | cuOpt Mean-CVaR notebook | RAPIDS + JupyterLab | 2 | ⏳ future | Same as single-cell |
| jax | JAX/marimo tutorial env | JAX | 2 | ⏳ future | marimo SPA + ws prefix risk; interactive |
| station-topic-modeling | BERTopic on 40M reviews | conda + Streamlit | 2 | ❌ | Station-targeted (≥64GB GPU); conda-native |
| station-rec-sys | Fashion recommender train + serve | HLLM + FastAPI | 2 | ❌ | Station-targeted; venv + flash-attn build |

### Dev tools / kernels

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| cutile-kernels | cuTile benchmark suite | CUDA devel | 3 | ❌ | Dev/benchmark tool, interactive container, no service value |
| station-kernel-dev-ft | Triton kernel dev + FT | NGC pytorch | 3 | ❌ | Interactive dev workflow, no service |
| station-nanochat | Pretrain→SFT→chat 1B model | NGC pytorch | 2 | ❌ | Station-targeted, 12h batch training |
| vibe-coding | VS Code + Continue on Ollama | host + systemd | 2 | ❌ policy | gpt-oss-120b exceeds budget (120B); engine policy; desktop client |
| vscode | VS Code install/remote | desktop GUI | 3 | ❌ | Desktop app, X11 |
| station-ai-skills | dgx-assist + agent skills | host CLI | 3 | ❌ | Host installer, no service |

### Connectivity / multi-node / infrastructure (host-level by nature)

| Playbook | What it does | Stack | Tier | Verdict | Blocker / workaround |
|----------|--------------|-------|------|---------|----------------------|
| connect-to-your-spark | SSH/NVIDIA Sync access | client tooling | 3 | ❌ | Connectivity tooling, not a workload |
| connect-two-sparks | 200GbE QSFP link + SSH | netplan + NICs | 3 | ❌ | Physical cabling, host net config |
| connect-three-sparks | Ring topology (3 nodes) | netplan + NICs | 3 | ❌ | Physical cabling, multi-node |
| multi-sparks-through-switch | 4+ Sparks via QSFP switch | switch + netplan | 3 | ❌ | External switch hardware |
| nccl | NCCL build + multi-node benchmarks | MPI + CX-7 | 3 | ❌ | Multi-node, sudo, host builds |
| tailscale | Mesh VPN access | host daemon | 3 | ❌ | Host network layer + SaaS account |
| register-to-brev | Brev managed remote access | SaaS agent | 3 | ❌ | Proprietary SaaS + host agent |
| isaac | Isaac Sim/Lab from source | host build + X11 | 3 | ❌ | Host build, GUI; headless container path exists but not the playbook |

### Station series (GB300-retargeted — out of scope for the Spark kit)

All `station-*` playbooks retarget base playbooks (or are Station-only) to DGX Station GB300
(252–288GB HBM3e, ARM64). None apply to the single-GPU DGX Spark beyond what the base playbook
already covers. Notable ones: station-vllm / station-sglang-inference (TIER-1 but Station-targeted
models — Kimi-K2.5 1T, DeepSeek-V4-Flash), station-mig (B300 MIG partitioning), station-brev
(SaaS), station-connect-two-stations (CX8 fabric), station-nemoclaw (352GB model, k3s sandbox),
station-nvfp4-pretraining (208GB VRAM), station-healthcare-agent (≥150GB VRAM).

## Future candidates (not this round)

| Candidate | Source | Work required |
|-----------|--------|---------------|
| Jupyter/RAPIDS workloads (3) | cuda-x-data-science, single-cell, portfolio-optimization | RAPIDS NGC image, `--ServerApp.base_url` or strip-sidecar + ws kernels, PVC persistence rework, publicAccess default false |
| `nvfp4-quantization-spark` (Job) | nvfp4-quantization | New Job-shaped plugin type (kit has no Job precedent); runtime git-clone+pip in NGC vLLM image; then serve artifact via existing vLLM archetype |
| `nemotron-nano-omni-spark` | nemotron | Custom image (`vllm[audio]`); multimodal-as-text note; memory budget re-derivation |
| Qwen3.6 NVFP4 alignment | vllm | Playbook recipe: util 0.4, ctx 262144, seqs 4, NVFP4 (~25GB) vs kit's qwen3-6-spark FP8 util 0.55/131072/4. Re-validation needed before touching a working config |
| Ollama / SGLang / TRT-LLM / NIM / llama.cpp | respective | Requires amending the vLLM-only engine policy decision record |

## Disqualified summary by category

- **Multi-node / physical**: connect-two-sparks, connect-three-sparks, multi-sparks-through-switch,
  nccl, station-connect-two-stations — host networking, cabling, SSH/MPI orchestration.
- **Host systemd / sudo installers**: hermes-agent, openclaw, openshell, nemoclaw,
  nemoclaw-applications, cli-coding-agent, ollama (host path), lm-studio, vss, tailscale,
  register-to-brev, station-ai-skills, station-local-coding-agent, station-brev.
- **Hardware / desktop GUI**: reachy-photo-booth, spark-reachy-photo-booth (USB robot),
  dgx-dashboard (firmware), vscode (X11), isaac (X11), rag-ai-workbench (desktop app).
- **WebRTC / origin-pinned / prefix-hostile**: live-vlm-webui (WebRTC UDP + HTTPS), nemoclaw /
  openshell / openclaw dashboards (token-in-fragment + `127.0.0.1` origin checks).
- **Batch-only / interactive-only**: multi-modal-inference, cutile-kernels, station-kernel-dev-ft,
  station-nvfp4-pretraining, station-gr00t, station-nanochat — no always-on service surface.
- **Memory budget violations**: gpt-oss-120b (vibe-coding, multi-agent-chatbot default — ~120GB
  FP8), Nemotron-3-Super-120B at util 0.9 (nemotron) — exceed the ~95–100GB clean budget.
- **Policy-skipped (engine)**: sglang, trt-llm, speculative-decoding, llama-cpp, ollama, nim-llm.
- **Station-only (GB300)**: station-vllm, station-sglang-inference, station-mig,
  station-nvfp4-pretraining, station-gr00t, station-kernel-dev-ft, station-nanochat,
  station-healthcare-agent, station-topic-modeling, station-rec-sys, station-txt2kg (partial —
  base txt2kg built), station-comfyui (partial — base comfy-ui built), station-openshell /
  station-nemoclaw / station-nvfp4-quantization (partial).

## Result

**3 built** (open-webui-spark, comfy-ui-spark, txt2kg-spark) · **5 already covered** (vLLM model
plugins) · **5 future candidates** · **53 disqualified or deferred** with the reasons above.
