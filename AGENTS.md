# AGENTS.md — Terra Spark Kit

Guidance for AI agents and automated tools working in this repository. Read before changing plugins or tooling.

> **Authoritative rules live upstream.** This repo is an extension of the official plugin catalog.
> Read [Terra-Official-Plugins/AGENTS.md](https://github.com/juno-fx/Terra-Official-Plugins/blob/main/AGENTS.md)
> for the complete rule set (plugin taxonomy, field types, workload template contract, Kuiper annotations).
> This file documents what is specific to Terra-Spark-Kit and summarizes the rules that matter most.

---

## Repository Purpose

This repository is a **Terra plugin Source** — a Git repo added via the Terra UI. Terra scans
`plugins/*/terra.yaml` and loads every plugin. All plugins live in `plugins/<plugin-name>/`.
Do not create plugins outside this directory.

Plugins here are **optimized for small devices like the NVIDIA DGX Spark** (Grace Blackwell, 128GB
unified memory, single GPU, single-user). Optimizations take the form of tuned defaults: nvidia
container runtime, GPU enabled by default, memory/utilization budgets sized for one device, minimal
resource overhead.

## Naming Convention

- Every plugin uses a distinct `-spark` resource ID (e.g. `ollama-spark`, `vllm-spark`)
- This prevents collisions with the official catalog when both Sources are enabled in Terra
- Never reuse an official plugin's `resource_id`

## README Catalog Convention

- Every plugin MUST have a `README.md`
- The root `README.md` MUST list every plugin in its Plugin Catalog table (Plugin, Type, Category,
  Description, Docs link)
- Adding or renaming a plugin REQUIRES updating the root README table in the same change

## Plugin Types

| Type | Marker | Install target |
|------|--------|----------------|
| Namespaced | no `cluster-level` tag | user's project namespace |
| Cluster-level | `cluster-level` tag | `argocd` namespace |
| Workload template | `cluster-level` tag + `templates/metadata.yaml` with `kuiper.juno-innovations.com/chart` label + `scripts/chart/` | `argocd` namespace |

## Critical Rules

1. **Repackage after changing `scripts/`** — `make package <plugin>` regenerates
   `templates/packaged-scripts.yaml` + `templates/packaged-scripts-cleanup.yaml`. Skipping it deploys
   stale scripts with no error. `make verify` detects staleness.
2. **1MiB ConfigMap limit** — `make check-size <plugin>` warns at 900KB, errors at 1MiB. Never add
   large binaries/media to `scripts/`.
3. **`metadata.yaml` is the workload template contract** — field names in `data.fields:` must exactly
   match keys in `scripts/chart/values.yaml` or Helm rendering fails at launch.
4. **Never edit generated files** — `packaged-scripts*.yaml` are always overwritten by `make package`.
5. **`terra.yaml` fields are install-time only** — shown in the Terra app store; they become Helm
   values at ArgoCD sync. Workload template fields live in `metadata.yaml`, shown at launch time.
6. **Ingress prefix pathing is REQUIRED** — every ingress in this kit MUST route
   `host: {{ .Values.host }}` with path `/plugin/<name>` (pathType Prefix, name `<name>-ingress`),
   plus `nginx.ingress.kubernetes.io/proxy-connect/read/send-timeout: "600"` (long LLM generations
   exceed the 60s nginx default). This mirrors the platform convention (`/vllm/<name>`,
   `/hermes/<name>`). If a workload's backend cannot serve requests under the prefixed path, ship an
   **nginx sidecar container** in the workload pod that strips the prefix
   (`rewrite ^/plugin/<name>(/|$)(.*) → /$2`, `proxy_pass` to `localhost:<app-port>`) — prefix
   support is never optional. The scaffold enforces this; do not regress it.

## Field Types

| Type | Where | Extra keys |
|------|-------|------------|
| `string`, `int`, `boolean` | both | `default` |
| `select`, `multi` | both | `options: [...]` |
| `shared-volume`, `exclusive-volume` | terra.yaml | — |
| `env`, `multi-line`, `list` | metadata.yaml | `list` has `fields: [...]`, no nested lists |
| `k8sPriority`, `k8sStorageClass`, `k8sIngressClass`, `k8sServiceAccount`, `dataVolume` | metadata.yaml | — |

## Engine Policy (decision record)

- **vLLM is the platform of record** for LLM serving in this kit. Every model plugin is a plain
  **vLLM StatefulSet** (single archetype) — no operator, no CRDs, no DGD/DGDR. Image tag:
  `vllm/vllm-openai:latest` for all models EXCEPT `nemotron-spark`, which pins `v0.27.1`
  (DSpark speculative decoding floor — its validated config; do not bump without re-validation).
- **Dynamo was rolled back** (decision record): the Dynamo Platform operator path crashed the
  GB10 node (NodeNotReady ×4 in 18h, all correlated with Dynamo-based model launches —
  operator children + frontend/worker processes + 112GB CUDA ceiling exceeded the unified-memory
  budget). Removed from kit + cluster. Re-add only when NVIDIA validates GB10 and Dynamo ships
  vLLM ≥ 0.27.1.
- **Memory rules (the node-killer fix)**: DGX Spark real budget ≈ 95–100GB (117.5Gi allocatable
  minus k8s/stack). Budgets are **footprint-derived per model** — KV pool = 2 × layers × kv_heads
  × head_dim × max_model_len × max_num_seqs (fp8 KV = 1B/elem), plus weights, activations, runtime:

  | Plugin | max_model_len | seqs | weights | KV pool | footprint | memory req=lim | util (ceiling) |
  |--------|--------------|------|---------|---------|-----------|----------------|----------------|
  | nemotron-spark | 131072 | 4 | ~18GB | ~14GB | ~39GB | 48Gi | 0.40 (51GB) |
  | nemotron-nano-spark | 131072 | 4 | ~17GB | ~14GB | ~38GB | 48Gi | 0.40 (51GB) |
  | qwen3-spark | 131072 | 2 | ~60GB | ~13GB | ~82GB | 88Gi | 0.70 (90GB) |
  | qwen3-6-spark | 131072 | 4 | ~35GB | ~21.5GB | ~64GB | 72Gi | 0.55 (70GB) |
  | nemotron-super-49b-spark | 32768 | 4 | ~50GB | ~21.5GB | ~79GB | 88Gi | 0.65 (83GB) |

  Container OOM (restart) instead of host OOM (node crash). 64K ctx × 4 seqs for super-49b would be
  ~101GB — does not fit; qwen3 at 4 seqs (~94GB) also exceeds the clean budget, hence seqs=2.
  gpt-oss-120b was removed from the kit because its FP8 load (~120GB) cannot fit.
- **cpu/memory fields (Guaranteed QoS)**: `cpu` (default `"2"`) and `memory` (per-model footprint
  defaults above) drive requests AND limits on the vLLM container (rendered `| quote`); the nginx
  sidecar carries 100m/128Mi req==limit so the pod is fully Guaranteed. Guaranteed reserves the
  full request — two model workloads can never co-schedule on the 117.5Gi allocatable node
  (48+48 and 88+96 > 117.5); one model at a time is the intended usage. CPU is intentionally
  minimal — flat 2 (req==lim) on the busy 20-core node; the exposed `cpu` field lifts per
  workload (heavy prefill may want 4).
- **Kuiper merge behavior** (user-verified): metadata field defaults override Kuiper's injected
  standard values (`cpu`/`memory`) at launch. Residual risk documented: if a future Kuiper changes
  merge order, requests silently render 1/1Gi — the per-launch requests check in validation
  (`kubectl get sts <name>` requests == "2"/"48Gi", "2"/"72Gi" or "2"/"88Gi" per model) catches it; fallback =
  rename the fields.
- **No per-model engine toggle.** One archetype per plugin (locked decision — see git history).
- **Model ingresses are PUBLIC by design** (no Hubble auth; user decision). The five `-spark`
  model workload ingresses ship hardcoded public — no auth-url annotation. The generic
  `template/workload` scaffold keeps its `publicAccess` toggle for non-model workloads. Host +
  `/plugin/<name>` prefix + 600s timeouts still required (Rule 6).
- **Nginx prefix-stripping sidecar is MANDATORY**: plain vLLM serves `/v1/*` at root and nginx
  passes URIs unchanged, so `/plugin/<name>/...` would 404 without it. Every workload pod carries
  an `nginx:alpine` sidecar (:8080) with `rewrite ^/plugin/<name>(/|$)(.*)$ /$2 break` +
  `proxy_pass http://127.0.0.1:8000` + 600s timeouts; Service + ingress target :8080.
- **Non-model companion workloads** (open-webui-spark, comfy-ui-spark, txt2kg-spark — ported from
  the NVIDIA DGX Spark playbook catalog; full scoping in `docs/playbook-scoping.md`):
  - Use the `publicAccess` toggle (Hubble auth-url when false), NOT the hardcoded-public model
    pattern. Defaults: open-webui/comfy-ui/txt2kg `false` — open-webui's app auth is DISABLED
    (WEBUI_AUTH=false, Hubble-only).
  - Same mandatory sidecar + `/plugin/<name>` prefix, but with the streaming additions: the
    `$connection_upgrade` map + `proxy_set_header Upgrade/Connection` (WebSocket), `proxy_buffering
    off` (SSE/progress), and `client_max_body_size 256m` for image uploads (comfy-ui + open-webui).
    comfy-ui's
    frontend serves the prefix natively (api_base from `window.location.pathname`, frontend ≥ 1.33)
    — sidecar only strips; a `location = /plugin/<name>` 301 keeps the base correct; no sub_filter.
    The 301 must run with `absolute_redirect off` — nginx's default would emit an absolute
    `http://host:<sidecar-port>/...` Location that browsers follow to the dead service port. Strip
    via `$request_uri` (raw/encoded), NOT `rewrite` — rewrite percent-decodes `%2F` and breaks
    `POST /api/userdata/{file}` saves (workflows → 405, Comfy-Org/ComfyUI#9664); exclude the
    query from the capture and re-append `$is_args$args` (avoids kaanlabs' filename-query bug).
  - **CPU-only companions** (open-webui, txt2kg): no GPU field, no `runtimeClassName` — they
    co-schedule with exactly one model workload (48Gi+4Gi and 88Gi+4Gi fit 117.5Gi). **comfy-ui
    is the GPU exception**: it requests `nvidia.com/gpu: 1` + `runtimeClassName: nvidia` and is
    mutually exclusive with model workloads (single-GPU node).
  - They reach the model via a **user-entered `llm_url` field** (`http://<model-workload-name>:8080/plugin/<model-workload-name>/v1`
    — the model sidecar's rewrite strips `/plugin/<name>`, so `/v1` reaches vLLM; a bare `/v1` path (no `/plugin/<name>` prefix) 404s
    (model sidecars have no default location). The model workload's served name is
    always `model` (txt2kg's `llm_model` default).
  - Bootstrap-heavy plugins (txt2kg) put their install/build logic in a ConfigMap script inside
    `scripts/chart/templates/`, mounted and run as the container `command` — the packaged-scripts
    `entrypoint.sh` is NOT executed for workload templates. comfy-ui needs no bootstrap: the
    image's own entrypoint provisions the venv on first boot.
  - **txt2kg-spark is the pass-through exception to the strip-sidecar**: Next.js can't serve
    under a stripped root (assets + client `/api` fetches are root-absolute), so the bootstrap
    bakes `basePath: /plugin/<name>` + prefixed fetches at build time and the sidecar proxies the
    URI unchanged (`proxy_pass` without rewrite). Prefix support stays mandatory — native serving
    under the prefix satisfies it. Rebuild marker `/workspace/.basepath` re-runs the build when
    the prefix changes.
  - **comfy-ui-spark** uses the DGX-native container `mmartial/comfyui-nvidia-docker`
    (`ubuntu24_cuda13.1-dgx-latest` — arm64, torch 2.10 cu13.1, Docker Hub anonymous pull; the
    NGC PyTorch base is 401-gated without a pull secret and fights torch downgrades). Non-root
    `comfy` user (uid/gid 1000) + pod `securityContext.fsGroup: 1000` for PVC writes; models are
    managed via the UI (no auto-download). txt2kg-spark fetches the playbook repo at a pinned
    commit and builds with pnpm at first boot (both persist on PVC, so first boot is slow,
    restarts are fast).
  - **Manager config.ini 3-key contract** (comfy-ui-spark): the mmartial entrypoint patches
    `user/__manager/config.ini` in place from envs (`SECURITY_LEVEL`, `USE_UV`, `NETWORK_MODE`)
    and then GREPS each key — a missing line kills the boot pre-patch (observed: 2-key seed →
    exit 1). The initContainer therefore seeds all THREE keys every boot
    (`security_level = weak`, `use_uv = true`, `network_mode = personal_cloud`) — locked config,
    Manager UI changes clobbered at restart. `network_mode = personal_cloud` is REQUIRED for
    Manager model downloads (the default `public` is denied by the security policy).
  - **comfy-ui-spark limits contract**: unlike model workloads (Guaranteed req==lim, locked),
    comfy renders limits ONLY from `cpuLimit`/`memoryLimit` when set — empty = no limit
    (platform scaffold contract; the chart previously forced limits=requests from `memory`,
    which OOM-killed renders at the request value, e.g. 8Gi). Burstable by default — a runaway
    render risks node memory pressure (~117.5Gi allocatable); cap via `memoryLimit` if needed.
  - **open-webui-spark limits contract + prefix strategy**: same Burstable limits contract
    (limits only from `cpuLimit`/`memoryLimit`, empty = no limit — prophylactic shift, no
    observed OOM). Prefix: Open WebUI has NO native subpath support upstream (PRs #23242/#15583
    unmerged; issues #25796/#17257), so unlike comfy the sidecar content-REWRITES: sub_filter
    rules (`/api/`, `/auth`, `/_app/`, `/static/`, `/ws/` — html/js/css only, never JSON),
    `proxy_redirect` ×3 (`/auth`, `/oidc`, `/sso`) for backend 302s, `Accept-Encoding ""` +
    buffered default location, unbuffered `^~ /ws/` + SSE regex locations
    (`/api/(v1/)?(chat/completions|retrieval/process)`). Two needle families: quote variants
    (`"/api/` etc.) for literal URL strings, and the `}/` family — SvelteKit builds many URLs as
    template literals with runtime-empty base vars (`` `${rn}/_app/version.json` ``,
    `` `${p}/api/config` ``), invisible to quote-needles; the `}` before `/path` is stable and
    rewritable (idempotent — rewritten output no longer matches). THE subpath fix: the inline
    hydration script sets `__sveltekit_n2lct2 = { base: "" }` — rewriting `base: ""` to
    `base: "/plugin/<name>"` makes the router resolve every route under the prefix (an empty
    base renders the app's own 404). Cautionary class: generic `/auth` needles were REMOVED —
    they corrupted the baked route manifest (`"/auth"` route id → prefixed, unmatchable after
    base strip). ALL navigation is fixed by patching the compiled
    `get_base_uri` (SvelteKit's goto resolves against document.baseURI — URL
    semantics make root-absolute paths override it, so every `goto('/x')` went
    bare): ONE one-shot needle rewrites `new URL(e,t)}function q()` to strip the
    leading slash (relative → prefixed) with an `indexOf("/plugin/")` guard for
    already-prefixed paths, resolving against the PREFIX ROOT
    (`location.origin+U+"/"`) — page-depth independent (subpage navigation
    can't double-prefix). nginx sub_filter rules do NOT chain (each matches
    the raw body) — never split a transform across chained needles. Completes
    the WEBUI_AUTH=false flow (guard → prefixed login → signin
    auto-authenticates admin@localhost). Native anchor hrefs (client-rendered,
    e.g. sidebar `/workspace`) bypass goto entirely — handled by
    `href:`-anchored needles enumerated from the live build.
    txt2kg stays Guaranteed until its limits edit is validated.
- **Deferred**: `nvidia-gpu-operator` stays in the official catalog (do not duplicate here).
- **Prerequisites** for all model plugins: official `nvidia-gpu-operator` plugin, k8s ≥ 1.30,
  host driver ≥ 580. NO Dynamo Platform required.
- **Field surface** (user-confirmed): `gpu_memory_utilization` + `hf_token` on ALL model plugins
  (required for NVIDIA-gated checkpoints; optional but always available for ungated models), plus
  `cpu` + `memory` (Guaranteed QoS requests, see memory rules) and `storage_class`
  (k8sStorageClass picker, REQUIRED) + `storage_size` (int, Gi) for the model cache
  PVC, plus `served_model_name` (default `"model"` — the OpenAI-compatible model string clients
  send; standardized across all kit workloads, each deployment has its own ingress). Context
  length, concurrency, model, checkpoint, quant, image and engine flags are LOCKED in
  `scripts/chart/values.yaml` (rendered with `| default` fallbacks).
- **Hardware validation pending**: configs are transcriptions of published NVIDIA/vLLM commands.
  No GB10 in CI. First real launches (nemotron-spark → nano → super-49b → qwen3, in that order)
  are the gate: node must stay Ready, worker Running & serving.

## Adding a New Model Plugin — Playbook

This is the proven end-to-end process for adding a model to the kit (used for nemotron-spark,
nemotron-nano-spark, qwen3-spark, nemotron-super-49b-spark, qwen3-6-spark). Follow it in order.

### 1. Research the model (all read-only, before any files)

1. **Check availability**: `curl -s "https://huggingface.co/api/models?author=<org>&search=<name>"` —
   the LISTING API is the reliable existence + gated check. Direct `/api/models/<id>` GETs return
   401 auth-wall for everything (gated or not) — useless as an existence probe. `gated: False`
   means no token needed; still ship the `hf_token` field (kit convention).
2. **Verify the exact repo ID**: HF repo names use underscores where NIM-style names use dots
   (e.g. `nvidia/Llama-3_3-Nemotron-Super-49B-v1_5-FP8`, NOT `Llama-3.3-...-v1.5-FP8`). A wrong ID
   = 404 at launch. Check the listing output, not the docs.
3. **License**: `curl -s "https://huggingface.co/api/models/<id>" | jq .cardData.license` (Apache-2.0
   is the kit baseline; anything else → flag in README).
4. **Architecture**: fetch `config.json` — quantized repos may block resolve; use the BASE model's
   config. Resolve redirects: `curl -sL -H "User-Agent: curl"` (307s otherwise). Record:
   `num_hidden_layers`, `num_key_value_heads`, `head_dim`, `max_position_embeddings`, experts/active
   (MoE). Multimodal checkpoints (`pipeline_tag: image-text-to-text`) work as text LLMs — note
   vision as untested in the README.
5. **Proven flags**: if the model runs anywhere (this cluster, vendor blogs, vLLM release notes),
   copy the exact flags. Reasoning/tool parsers are model-family-specific (`qwen3` for Qwen3.6,
   `deepseek_r1` for Qwen3-2507/Nano, `nemotron_v3` for Lightning) — never assume from a sibling
   model.
6. **Memory math** (the budget table): footprint = weights + KV pool + activations (~3–5GB) +
   runtime (~3GB). KV pool = 2 × layers × kv_heads × head_dim × max_model_len × max_num_seqs,
   fp8 KV = 1 byte/elem. Derive: `memory` req=lim ≥ footprint (Guaranteed containment);
   `gpu_memory_utilization` ceiling = 128GB × util ≥ CUDA portion (weights + KV + activations)
   with ~5–10GB margin; the ceiling must sit BELOW the cgroup limit. If the numbers don't fit,
   reduce ctx or max_num_seqs (locked) — never exceed the budget. One model at a time is the
   intended usage (48+48 and 72+88 > 117.5Gi allocatable).

### 2. Build the plugin (clone an existing archetype-A model plugin)

1. **Resource ID must be DNS-1035-safe** — lowercase alphanumerics + hyphens ONLY. No dots:
   `qwen3.6-spark` was rejected by `helm lint` (k8s object names). Use `qwen3-6-spark`.
2. Clone the closest existing plugin (e.g. `cp -r plugins/qwen3-spark plugins/<new-id>`) and sweep
   every file for the old ID/name (sed). Update: terra.yaml (`resource_id`, `name` display,
   description, tags), Chart.yaml ×2 (name + description), values.yaml (`name` key + icon),
   metadata.yaml (`data.description` + icon default), README.
3. Set the locked config in `scripts/chart/values.yaml`: `model` (verified ID), image
   `vllm/vllm-openai:latest`, ctx/seqs/memory/util per the budget table, `runtimeClassName: nvidia`.
4. Set the args in `scripts/chart/templates/statefulset.yaml` (flags live HERE, not values.yaml):
   kv-cache fp8, prefix caching, tool/reasoning parsers per the model family, plus the probe trio
   (startup `/health` threshold 120, liveness `/health` threshold 3, readiness `/v1/models`
   threshold 10), nginx sidecar (:8080, 100m/128Mi), memory limit. Update the `| default`
   fallbacks for max_model_len/max_num_seqs.
5. Metadata fields (metadata.yaml, defaults match values.yaml — Kuiper injects these at launch):
   `icon`, `served_model_name` ("model"), `cpu` ("2"), `memory` (footprint value + description),
   `gpu_memory_utilization` (ceiling value + description), `hf_token` (optional), `storage_class`
   (REQUIRED), `storage_size` (100). Keep the dual-defaults parity (metadata default == values key).
6. Copy `assets/logo.png`; verify the icon URL across ALL 5 surfaces (terra.yaml, root Chart.yaml,
   scripts/chart/Chart.yaml, values.yaml, metadata default) — identical string.
7. README: model facts, footprint table, config table, fields, prerequisites, confidence notes
   (assembled vs published configs), multimodal-as-text note if applicable.

### 3. Catalog sweep (same change)

- `bundles/spark-llm.yaml`: add the entry; update the description count (n workloads).
- Root `README.md`: add the catalog row; sweep "four/five model workloads" count; update the
  per-model budget enumeration line (48Gi Lightning/Nano, 72Gi Qwen3.6, 88Gi Qwen3/Super-49B;
  util 0.40–0.70).
- `AGENTS.md`: add the row to the memory-rules table; extend the Kuiper-merge validation check
  string (`requests == "2"/"48Gi", "2"/"72Gi" or "2"/"88Gi"`); update "The N -spark model
  workload ingresses" count.

### 4. Gates (all must pass)

1. `helm template` render: model arg, parsers, ctx/seqs/memory/util quoted (`"131072"`, `"72Gi"`,
   `"0.55"`), requests == limits, no DynamoGraphDeployment, probes present.
2. Parity: every metadata field default == values.yaml value (cpu, memory, gpu_memory_utilization,
   served_model_name, icon, storage_size).
3. `make package <id>` (auto check-size) → `make verify` → `make lint` (10 charts for 5 plugins).
4. No `*.tar`/`*.tgz`/`*.base64` leftovers.

### 5. Validate on the cluster (the launch ladder)

1. Commit + push → ArgoCD sync → the terra-metadata + scripts ConfigMaps must show the new defaults
   (`kubectl get cm <id>-argocd-terra-metadata -n argocd`).
2. Launch from Genesis. First boot: model download (local-path) → load → AutoTuner (CPU-threaded —
   at 2 CPU it's slow; the 20-min startup budget covers it) → readiness flips only when `/v1/models`
   answers.
3. Verify: `kubectl get pod <name>-0` = 2/2, restarts 0; requests read `2`/`<mem>`;
   `curl https://tdk.hatfieldfx.com/plugin/<name>/v1/models` → 200, `id: "model"`,
   `max_model_len` = the configured ctx; node stays Ready.
4. Known failure signatures: exit 2 fast = removed flag on `:latest` drift (drop/update the flag);
   CrashLoopBackOff 401 = HF_TOKEN wiring; hung_task panics = check `/etc/cron.d/drop-caches`
   (already removed — never re-add).

## Development Environment

`devbox shell` is a prerequisite for all make targets (helm, kind, kubectl, docker). Run
`make new-plugin` from inside the shell.

## Make Targets

| Target | Usage |
|--------|-------|
| `make new-plugin` | interactive scaffolding (namespaced / cluster / workload) |
| `make package <name>` | package scripts/ into ConfigMap |
| `make verify` | stale-package check (CI runs this) |
| `make check-size <name>` | 1MiB limit check |
| `make watch <name>` | auto-repackage on scripts/ changes |
| `make lint` | helm lint --strict all charts |
| `make test <name>` | deploy to local Kind cluster via ArgoCD |

## Known Quirks

- **Numeric field values arrive as typed YAML**: Kuiper passes user-entered values from the Genesis
  UI as typed YAML — `gpu_memory_utilization=0.6` arrives as a float, not a string. Kubernetes
  rejects non-string container args, so any arg rendered from a user field MUST use `| quote`
  (e.g. `{{ .Values.gpu_memory_utilization | quote }}`). Chart defaults in `values.yaml` are quoted
  strings and mask this — only user-entered numeric values hit it. First surfaced at launch:
  `args[35]: expected string, got value{Value:0.6}`.
- **`envFrom` exposes secret KEYS as env var names**: the `hf_token` secret key MUST be named
  `HF_TOKEN` (not `token`) — Dynamo's model downloader (hf-hub) and vLLM read `HF_TOKEN`, and
  `envFrom` cannot rename keys. Failure signature: worker CrashLoopBackOff with
  `401 Unauthorized` on a gated HuggingFace checkpoint.
- **`:latest` drift breaks flags**: model images use `vllm/vllm-openai:latest` by policy (user
  decision — no tag pinning). vLLM removes flags upstream without notice; first casualty:
  `--enable-reasoning` (crash signature: `vllm: error: unrecognized arguments` + exit 2 at
  startup). Fix pattern: drop/update the flag, repackage, relaunch. When a launch exits 2 fast,
  the first suspect is a removed flag on the drifted image.
- **Node panics were a cron, not the kit**: the DGX Spark node's repeated
  `Kernel panic - not syncing: hung_task` (5-min cycles) were caused by a root cron
  (`/etc/cron.d/drop-caches`: `sync; echo 3 > /proc/sys/vm/drop_caches`) wedging writeback past the
  61s hung_task threshold under I/O load. Removed. If hung_task panics ever recur, check for
  scheduled `sync` jobs before touching memory config.

## Adding a New Plugin — Checklist

1. `make new-plugin` — interactive prompts
2. Edit `terra.yaml` — `name`, `description`, `category`, `icon`, `fields`; resource ID must end in `-spark`
3. Edit `Chart.yaml` — bump version if needed
4. Add Kubernetes objects per plugin type (see upstream AGENTS.md for details) — ingress MUST use
   `host: {{ .Values.host }}` + `/plugin/<name>` prefix pathing + 600s proxy timeouts (Rule 6);
   if the backend cannot serve prefixed paths, add the nginx prefix-stripping sidecar
5. If plugin has `scripts/`: `make package <plugin-name>` — **required**
6. `make check-size <plugin-name>` — verify under 1MiB
7. Add README.md to the plugin
8. Add the plugin row to the root README catalog table
9. `make verify` — confirm nothing is stale
10. `make lint` — confirm charts pass
11. Commit `scripts/` changes AND regenerated `packaged-scripts*.yaml` together
