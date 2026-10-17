# combine

Combine front-end for an already-running model workload (`<model-plugin-id>`).
It renders **no pod of its own** — only a Service that selects the model's running pods, plus
an optional public Ingress.

Launch the model first, then launch this to expose it as Ingress, NodePort or LoadBalancer.

## What it creates

| Object | Detail |
|--------|--------|
| Object | Detail |
|--------|--------|
| Service `<name>` | selector `juno-innovations.com/model: <required>` → `targetPort: nginx`, the model pod's prefix-stripping sidecar on 8080 |
| Service type | `ClusterIP` (exposure `Ingress`), `NodePort`, or `LoadBalancer` |
| Ingress `<name>-ingress` | **Ingress mode only** — public, `host: {{ .Values.host }}`, path `/{{ .Release.Namespace }}/<name>`, `pathType: Prefix`, 600s proxy timeouts |
| Pods | none — the target model workload is untouched |

The prefix strip happens in the **model pod's nginx sidecar**, which carries a generic
`/{{ .Release.Namespace }}/` route next to its own `/{{ .Release.Namespace }}/<model>` route. The Ingress passes the path through
unchanged (no rewrite annotations), so the kit ingress convention holds.

## Fields

- `exposure` (default `Ingress`) — `Ingress` = ClusterIP Service + public Ingress on
  `/{{ .Release.Namespace }}/<name>`. `NodePort` and `LoadBalancer` create **no Ingress**; LoadBalancer is
  untested (this cluster has no load-balancer controller) and is out of scope for validation.
- `service_port` (default 8080) — the Service port, mapped to the pod's `nginx` port.
- `ingress_class` (default `nginx`) — IngressClass for the unify Ingress (Ingress mode only).
- `node_port` (default 0 = auto) — fixed NodePort (30000–32767) for NodePort/LoadBalancer modes;
  must be free cluster-wide or the Service is rejected.
- `icon` — display icon.

## Endpoints

Ingress mode:

```
https://<host>/{{ .Release.Namespace }}/<name>/v1/models
```

NodePort mode (the nodePort is what clients dial — `service_port` is the in-cluster port):

```
http://<node-ip>:<nodePort>/{{ .Release.Namespace }}/<name>/v1/models
```

The `/{{ .Release.Namespace }}/<name>` prefix is **required** in both modes: the sidecar strips it before
proxying to vLLM on `:8000`. A bare prefix with no `/v1/...` path resolves to `/`, which vLLM
answers with 404.

## Requirements and failure modes (silent 503)

- The model workload must be **running** and launched as the plugin id configured in the `model`
  field: the selector is driven by this field, so a differently named/modelled workload matches
  nothing → empty endpoints → 503.
- The companion must live in the **same project/namespace** as the model — Service selectors do not
  cross namespaces.
- The model pod must be (re)launched after the kit update that added the `/{{ .Release.Namespace }}/combine/` sidecar
  route; the sidecar does not hot-reload its ConfigMap.
- Verify: `kubectl get endpoints <name>` must resolve port **8080** (not 8000, not empty).

## Prerequisites

- A running workload with the matching plugin id in the same project.
- Ingress mode: the `nginx` IngressClass (platform default).
