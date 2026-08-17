# Terra-Spark-Kit

A series of [Terra](https://juno-fx.github.io/Orion-Documentation/) plugins optimized for small devices
like the NVIDIA DGX Spark. Each plugin is a Helm chart that Terra installs into Kubernetes clusters via ArgoCD.

This repository is a Terra plugin **Source** — add it in the Terra UI like any other source, alongside
or instead of the [official plugin catalog](https://github.com/juno-fx/Terra-Official-Plugins).
Plugins use distinct `-spark` resource IDs so they coexist cleanly with their official counterparts.

---

## Plugin Catalog

| Plugin | Type | Category | Description | Docs |
|--------|------|----------|-------------|------|
| Nemotron 3.5 Lightning (Spark) | Workload Template | AI | 30B-A3B NVFP4 on plain vLLM (pinned v0.27.1) with DSpark speculative decoding — NVIDIA's published DGX Spark config | [README](plugins/nemotron-spark/README.md) |
| Nemotron 3 Nano (Spark) | Workload Template | AI | 30B-A3B NVFP4 (QAD) on plain vLLM — official Spark playbook model | [README](plugins/nemotron-nano-spark/README.md) |
| Qwen3 30B-A3B (Spark) | Workload Template | AI | Qwen3-30B-A3B-Instruct-2507 BF16 on plain vLLM, 128K context | [README](plugins/qwen3-spark/README.md) |
| Qwen3.6 35B-A3B (Spark) | Workload Template | AI | Qwen3.6-35B-A3B-FP8 on plain vLLM, 128K context, 4 seqs | [README](plugins/qwen3-6-spark/README.md) |
| Nemotron Super 49B (Spark) | Workload Template | AI | Llama-3_3-Nemotron-Super-49B-v1_5-FP8 on plain vLLM | [README](plugins/nemotron-super-49b-spark/README.md) |
| Open WebUI (Spark) | Workload Template | AI | Chat UI pointed at any kit model workload — CPU-only companion, co-schedules with one model | [README](plugins/open-webui-spark/README.md) |
| ComfyUI (Spark) | Workload Template | AI | Node-graph image generation (SDXL/Flux) — GPU-exclusive workload, DGX-native container, models managed via the UI | [README](plugins/comfy-ui-spark/README.md) |
| txt2kg (Spark) | Workload Template | AI | Text-to-knowledge-graph (Next.js + ArangoDB) backed by any kit model workload — CPU-only companion | [README](plugins/txt2kg-spark/README.md) |
| Qwen3.8-27B (Spark) | Workload Template | AI | Qwen/Qwen3.8-27B — latest 27B model, tuned for the DGX Spark (FP8) | [README](plugins/qwen3-8-27b-spark/README.md) |

Bundles: [Spark LLM Stack](bundles/spark-llm.yaml) — all five model workloads in one install ·
[Spark AI Lab](bundles/spark-ai-lab.yaml) — Open WebUI + ComfyUI + txt2kg companions.

Every workload is a plain-vLLM StatefulSet with a footprint-derived unified-memory budget
(memory request = limit per model: 48Gi Lightning/Nano, 72Gi Qwen3.6, 88Gi Qwen3/Super-49B; util 0.40–0.70),
served under the platform `/plugin/<name>` prefix via an in-pod nginx prefix-stripping sidecar.

Three **non-model companion workloads** (Open WebUI, ComfyUI, txt2kg) follow the same ingress rule.
Model workloads are GPU-exclusive (one at a time); the CPU-only companions co-schedule alongside
any one model (48Gi+4Gi and 88Gi+4Gi both fit the 117.5Gi allocatable node).

For the full mapping of the NVIDIA DGX Spark playbook catalog to this kit (66 playbooks scoped,
tiers, disqualifications), see [docs/playbook-scoping.md](docs/playbook-scoping.md).

---

## Adding as a Terra Source

1. Push this repository to GitHub
2. In the Terra UI, add a new Source pointing at this repository URL
3. Terra scans `plugins/*/terra.yaml` and loads every plugin automatically

---

## Development

```bash
# 1. Enter the dev environment (required for all make targets)
devbox shell

# 2. Create a new plugin (interactive — prompts for type)
make new-plugin

# 3. Edit your plugin files

# 4. If your plugin has a scripts/ directory, package it
make package <plugin-name>

# 5. Verify nothing is stale
make verify
make lint
```

## Key Commands

| Command | Description |
|---------|-------------|
| `make new-plugin` | Interactive scaffolding for namespaced / cluster / workload plugins |
| `make package <name>` | **Required** after any `scripts/` change — bundles scripts into a ConfigMap |
| `make verify` | Checks all plugins have up-to-date packages (runs in CI on every push) |
| `make check-size <name>` | Checks packaged size against the 1MiB Kubernetes ConfigMap limit |
| `make watch <name>` | Auto-repackages when `scripts/` changes |
| `make lint` | Helm lint all plugins |
| `make test <name>` | Deploys plugin to a local Kind cluster with ArgoCD |
| `make down` | Destroys the local Kind cluster |

## The Packaging Rule

> **If you change anything in `scripts/` or `scripts/chart/`, you MUST run `make package <plugin-name>`.**
> If you skip repackaging, the old version deploys silently with no error. `make verify` catches this.

See [AGENTS.md](AGENTS.md) for the full rules. For authoritative upstream guidance, read the
[Terra-Official-Plugins AGENTS.md](https://github.com/juno-fx/Terra-Official-Plugins/blob/main/AGENTS.md).

## License

MIT — see [LICENSE](LICENSE).
