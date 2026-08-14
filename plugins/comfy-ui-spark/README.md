# comfy-ui-spark

**ComfyUI** node-graph AI image generation for the DGX Spark — **GPU-exclusive**: run instead of,
not alongside, a model workload. Source playbook:
[NVIDIA/dgx-spark-playbooks — comfy-ui](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/comfy-ui).

Runs the **DGX-native container** [`mmartial/comfyui-nvidia-docker`](https://github.com/mmartial/ComfyUI-Nvidia-Docker)
(`ubuntu24_cuda13.1-dgx-latest` — arm64 build, Ubuntu 24.04 + CUDA 13.1 + torch 2.10, sm_121-capable,
Docker Hub anonymous pull, maintained for this exact hardware; see the
[NVIDIA forum thread](https://forums.developer.nvidia.com/t/comfyui-container-for-dgx-spark/363342)).
The image's entrypoint provisions the venv on first boot, then runs ComfyUI — no custom bootstrap.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Image | `mmartial/comfyui-nvidia-docker:ubuntu24_cuda13.1-dgx-latest` |
| App port | 8188, served through the nginx prefix-stripping sidecar on :8080 |
| User | non-root `comfy` user (uid/gid 1000); pod `fsGroup: 1000` gives group-write on the PVC (the entrypoint does not chown mounted paths) |
| Security | `SECURITY_LEVEL=weak` (ComfyUI-Manager installs without prompts — arbitrary custom-node code, keep Hubble in front) |
| Manager config | `config.ini` seeded by the initContainer every boot with `security_level = weak` / `use_uv = true` / `network_mode = personal_cloud` — the **3-key contract** the mmartial entrypoint greps (a missing key kills the boot pre-patch); `network_mode = personal_cloud` is REQUIRED for Manager model downloads (default `public` is denied). Locked: Manager UI changes clobbered at restart (see AGENTS.md) |
| Launch flags | `COMFY_CMDLINE_EXTRA=--enable-manager-legacy-ui` (legacy Manager UI in the sidebar) |
| Storage | PVC `<name>-models` → `/basedir` (models, input, output, custom nodes) + subPath `run` → `/comfy/mnt` (venv, persists across restarts) |
| Streaming | Sidecar adds WebSocket upgrade headers (`/ws` protocol) + `proxy_buffering off` + `client_max_body_size 256m` |
| GPU | 1 × `nvidia.com/gpu`, runtime class `nvidia` |
| Memory | **Request** 48Gi default (SDXL/Flux-class; raise for video-class). **Limits are EMPTY by default — no CPU/memory limit** (platform contract: `cpuLimit`/`memoryLimit` at launch, empty = uncapped). Burstable QoS — a runaway render can push the node toward memory pressure (~117.5Gi allocatable); cap via `memoryLimit` if that bites |

Ingress: `/plugin/<name>` prefix; `publicAccess` default **false** (no app-level auth →
Hubble in front).

## Prefix routing — native subpath, no content rewriting

ComfyUI's frontend has served under a path prefix natively since ComfyUI_frontend 1.33
(Dec 2025 — [PR #7115](https://github.com/Comfy-Org/ComfyUI_frontend/pull/7115), extended by
[PR #12833](https://github.com/Comfy-Org/ComfyUI_frontend/pull/12833)): it derives its API base
from `window.location.pathname` and emits prefixed URLs itself (`/plugin/<name>/api/*`,
`/plugin/<name>/ws`, `/plugin/<name>/internal/*`). The sidecar therefore only strips the prefix
back to root (`rewrite ^/plugin/<name>(/|$)(.*)$ /$2 break;` + `proxy_pass
http://127.0.0.1:8188`) — **no `sub_filter` content rewriting, no source patches**. The mmartial
image clones latest ComfyUI at first boot, so the frontend always exceeds the 1.33 floor.

Two sidecar details matter:

- `location = /plugin/<name>` returns **301 to the trailing-slash form** — without it the
  frontend computes `api_base = "/plugin"` and nothing resolves. The sidecar sets
  `absolute_redirect off` so the 301 Location is RELATIVE: nginx's default (on) would emit an
  absolute `http://host:8080/plugin/<name>/` built from the sidecar's own socket — the browser
  follows it to the unfirewalled service port and times out.
- The classic `rewrite ... break` keeps query strings proper — workflow saves (`overwrite`,
  `full_info` params) work untouched. Community workarounds that embed the query string into the
  path (kaanlabs `$request_uri` trick; the issue-8325 comment configs) are obsolete here.

Known edge — workflow saves with subdirectories (issue 9664, **confirmed on this deployment**):
nginx `rewrite` percent-decodes `%2F`, so `POST /api/userdata/workflows%2Ffile.json` arrived as a
2-segment path → 405. Fixed in the sidecar by stripping via `$request_uri` (raw/encoded — `%2F`
survives; aiohttp decodes `match_info` after routing) with the query excluded from the capture
(`[^?]*`) and re-appended via `$is_args$args` — no ComfyUI source patches needed (this also avoids
the kaanlabs `$request_uri`-query-embedding bug). Fallback if a future pinned/old boot ever ships
a pre-1.33 frontend: nginx `sub_filter` content-rewrite of the bundle (`/assets`, `/api`, `/view`,
`/ws`).

## Fields

- `cpu` / `memory` — requests only (Burstable; limits empty by default), defaults 2 CPU / 48Gi.
- `cpuLimit` / `memoryLimit` — optional launch-time caps (Kuiper standard values, empty = no limit).
- `publicAccess` (default `false`).
- `storage_class` (required) / `storage_size` (default 100Gi) — checkpoints, loras, vae,
  output, custom nodes + the runtime venv. `local-path`/longhorn recommended.

## First boot (10–20 min)

1. Image entrypoint provisions the venv into `/comfy/mnt` (torch 2.10 cu13.1 + ComfyUI deps) —
   once; later boots are fast (venv persists on the PVC).
2. ComfyUI up when `/system_stats` answers (startup probe budget: 40 min).

## Managing models

No auto-download — place checkpoints via the UI (ComfyUI-Manager or drag-drop) or
`kubectl cp` into `/basedir/models/checkpoints` (also loras/vae/embeddings). Everything persists
on the PVC.

## Notes

- The playbook's OOM workaround (`sudo drop_caches`) is **not** replicated — container OOM
  containment via an optional `memoryLimit` cap instead (empty = uncapped Burstable; see AGENTS.md
  node-panic history).
- `SECURITY_LEVEL=weak` mirrors the upstream DGX compose default — custom nodes run arbitrary
  code; keep `publicAccess: false`.
- SageAttention / onnxruntime userscript extras (upstream `userscripts_dir`) are **not** wired in
  this kit — attention falls back to PyTorch; possible future perf work.
- Image tags follow `-latest` (kit policy): relaunching later picks up newer upstream builds.
- **Hardware validation pending** — assembled from the upstream DGX container docs; not yet
  launch-validated on this cluster.
