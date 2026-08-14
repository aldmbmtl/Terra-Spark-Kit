# open-webui-spark

**Open WebUI** chat frontend for the DGX Spark — a CPU-only companion workload that points at any
kit vLLM model workload. Source playbook:
[NVIDIA/dgx-spark-playbooks — open-webui](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/open-webui).

Unlike the playbook's bundled-Ollama image, this plugin uses the **frontend-only** image
(`ghcr.io/open-webui/open-webui:main`) — LLM serving stays on the kit's vLLM model plugins
(engine policy). No GPU, no runtime class: it co-schedules with exactly one model workload
(48Gi + 4Gi and 88Gi + 4Gi both fit the 117.5Gi allocatable node).

## Configuration (locked)

| Setting | Value |
|---------|-------|
| Image | `ghcr.io/open-webui/open-webui:main` (frontend-only — no bundled Ollama) |
| App port | 8090 (env `PORT`), served through the nginx prefix-stripping sidecar on :8080 |
| Storage | PVC `<name>-data` → `/app/backend/data` (SQLite chat history) |
| Streaming | Sidecar: WebSocket upgrade headers + `proxy_buffering off` on `/ws/` and the SSE paths only — the default location is BUFFERED (sub_filter requires it) |
| GPU | none — CPU-only companion |

Ingress: `/plugin/<name>` prefix, `publicAccess` toggle (default **false** — app auth is DISABLED
(`WEBUI_AUTH=false`), Hubble at the ingress is the only gate).

## Prefix routing — sub_filter strategy

Open WebUI has **no native subpath support** upstream (PRs #23242/#15583 unmerged; issues
#25796/#17257): the stock build emits root-absolute URLs (`/api/*`, `/auth`, `/_app/*`,
`/static/*`, `/ws/*`). Unlike comfy-ui (native subpath frontend), the sidecar therefore
**content-rewrites** the frontend's responses under `/plugin/<name>`:

- **Buffered default location** with `sub_filter` rules (quote variants `"/api/`, `'/api/`, `"/auth`,
  `"/_app/`, `"/static/`, `"/ws/`, `'/ws/`; the **`}/` family** for SvelteKit template-literal
  constructions — `${var}/_app/version.json`, `${var}/api/config` etc. are invisible to
  quote-needles, but the `}` before `/path` is stable; plus `"/manifest.json` and unquoted CSS
  `url(/static/` — html/js/css only, **never JSON** — rewriting JSON would corrupt stored chat
  content) + `Accept-Encoding ""` (sub_filter needs identity). **THE subpath fix**: the inline
  hydration script sets `__sveltekit_n2lct2 = { base: "" }` — rewritten to
  `base: "/plugin/<name>"`, making the SvelteKit router resolve every route under the prefix
  (empty base = the app's own 404 page)
- `proxy_redirect` ×3 (`/auth`, `/oidc`, `/sso`) — sub_filter can't touch backend 302 Location
  headers
- **Unbuffered** `^~ /plugin/<name>/ws/` (socket.io, upgrade headers) and the SSE regex location
  (`/api/(v1/)?(chat/completions|retrieval/process)`) — the default is buffered because
  sub_filter requires it
- `client_max_body_size 256m` (sidecar) + `proxy-body-size: 256m` (ingress) — uploads would
  otherwise 413 at the ingress's default 1m cap

The plain `rewrite` strip is safe here: Open WebUI URLs carry no `%2F`/embedded-filename paths
(UUID ids, not names) — the ComfyUI#9664 class doesn't apply. `:main` tag drift may change the
frontend's path strings — if the UI shows 404s on uncovered absolute paths, add sub_filter rules
(same iterate loop as the launch validation). **Do NOT add generic `/auth` needles** — they corrupt the baked SvelteKit route manifest (route
id `"/auth"` becomes `/plugin/<name>/auth`, which the router can never match after stripping
base). ALL navigation is fixed by patching the compiled `get_base_uri` (`new URL(e,t)}function
q()` → strip leading slash with an `indexOf("/plugin/")` guard for already-prefixed paths) —
SvelteKit's goto resolves against `document.baseURI`, where root-absolute paths always override
the document path; the patch makes them relative → prefixed. This supersedes the earlier
`?redirect=` needles (they double-prefixed under the patch). Completes the `WEBUI_AUTH=false`
flow: guard → prefixed login page → signin auto-authenticates `admin@localhost` → chat UI.

## Fields

- `llm_url` — OpenAI-compatible base URL of a **launched** kit model workload:
  `http://<model-workload-name>:8080/plugin/<model-workload-name>/v1` (the model sidecar's rewrite strips `/plugin/<name>`; `/v1` reaches vLLM — a bare
  unmodified). Launch a model first, enter its workload name. If empty, add a connection in the
  Open WebUI admin panel instead.
- `llm_api_key` (optional, sensitive) — key the model endpoint requires; kit model workloads do
  not require one.
- `cpu` / `memory` — requests only (limits from `cpuLimit`/`memoryLimit`, empty = no limit), defaults 1 CPU / 4Gi.
- `publicAccess` (default `false` — Hubble required; app auth disabled).
- `storage_class` (required) / `storage_size` (default 10Gi) — chat history volume.
  `local-path` recommended.

## Usage

1. Launch a model workload (e.g. `nemotron-nano-spark`), note its workload name.
2. Launch this plugin; set `llm_url` to `http://<model-workload-name>:8080/plugin/<model-workload-name>/v1`.
3. First boot: register the local admin account, then chat.

## Prerequisites

- Any kit model workload for the backend (no GPU required for this plugin itself).
- **Hardware validation pending** — assembled from the playbook + TRT-LLM playbook's
  frontend-only wiring (`OPENAI_API_BASE_URL`); not yet launch-validated.
