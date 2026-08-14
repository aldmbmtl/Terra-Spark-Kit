# txt2kg-spark

**txt2kg** — text → interactive knowledge graph (Next.js + Three.js WebGPU UI, ArangoDB storage,
LLM triple extraction). Source playbook:
[NVIDIA/dgx-spark-playbooks — txt2kg](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/txt2kg).

Kit adaptation: the playbook's Ollama backend is replaced by **any kit vLLM model workload**
(the playbook's own `--vllm` variant; its default model `Llama-3_3-Nemotron-Super-49B-v1_5-FP8`
is exactly what `nemotron-super-49b-spark` serves — under the kit's served name `model`).
CPU-only: this plugin co-schedules with one model workload.

## Pod layout (3 containers)

| Container | Image | Role |
|-----------|-------|------|
| `app` | `node:20-alpine` | Next.js app (:3000) — source fetched at pinned playbook commit, `pnpm install` + `next build` on first boot, persisted on the data PVC |
| `arangodb` | `arangodb:3.12` | graph DB (:8529), `ARANGO_NO_AUTH=1` (playbook default — Hubble auth in front); starts `arangod` + creates the `txt2kg` database in one shot (k8s init containers run before main containers, so a separate init cannot reach arangod) |
| `nginx` | `nginx:alpine` | **pass-through** sidecar :8080 → app :3000 (ws + buffering off) — no prefix strip |

## Prefix strategy (pass-through, not strip)

The Next.js app **serves natively under the prefix**: the bootstrap bakes
`basePath: /plugin/<name>` into `next.config.mjs` and prefixes all client-side
`fetch('/api/...')` calls at build time (the rebuild marker `/workspace/.basepath` re-runs the
build if the prefix ever changes). The sidecar therefore passes URIs through unmodified — unlike
the model plugins / root-relative SPAs, where stripping is required.

## Configuration (locked)

| Setting | Value |
|---------|-------|
| App source | playbook repo pinned `1fb66f0…`, `assets/frontend` |
| Graph DB | ArangoDB, database `txt2kg`, `ARANGO_NO_AUTH=1` |
| LLM | external via `llm_url` (kit vLLM workload); `VLLM_MODEL=model` (kit served name); Ollama disabled |
| Vector search (Qdrant) | deferred — not shipped; triples + graph only |
| GPU | none — CPU-only companion |
| Resources | app 2 CPU / 8Gi, arangodb 500m / 2Gi, nginx 100m / 128Mi — all req = lim (pod Guaranteed) |

Ingress: `/plugin/<name>` prefix; `publicAccess` default **false** (no app auth, `ARANGO_NO_AUTH=1`
→ Hubble in front).

## Fields

- `llm_url` (**required**) — OpenAI-compatible base URL of a launched kit model workload:
  `http://<model-workload-name>:8080/plugin/<model-workload-name>/v1`. Launch a model first (playbook pairing:
  `nemotron-super-49b-spark`).
- `llm_model` (default `model`) — must match the model workload's `served_model_name`.
- `cpu` / `memory` — Guaranteed QoS defaults 2 CPU / 8Gi (app container; arangodb + sidecar fixed
  on top).
- `publicAccess` (default `false`).
- `storage_class` (required) / `storage_size` (default 50Gi) — ArangoDB data + app workspace
  (node_modules/.next, built once).

## First boot (10–15 min)

1. ArangoDB container starts `arangod` and creates the `txt2kg` database (idempotent; readiness gate via `/_api/version`).
2. App container: fetches pinned playbook source, `pnpm install`, `next build` (once; persisted).
3. Ready when Next.js answers `/` (startup probe budget: 20 min).

## Notes

- Playbook's full stack also runs Ollama-in-compose + optional Qdrant embeddings — omitted by
  design (engine policy; scope: triples + graph).
- `ARANGO_NO_AUTH=1` is the playbook default — do not expose publicly without Hubble
  (`publicAccess: false`).
- **Hardware validation pending** — assembled from playbook compose + Dockerfile, not yet
  launch-validated.
