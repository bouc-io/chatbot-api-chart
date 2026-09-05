# chatbot-api-chart

Helm chart for `chatbot-api-server`, the conversational backend that preceded the agent platform.

> **This service is legacy.** `agent-api-server` and `agent-api-chart` supersede it, and new work
> belongs there. This chart is kept running and maintained, but it should not be the starting point
> for anything new.

## What it deploys

A stateless Deployment plus a ClusterIP Service, backed by one bundled subchart: **PostgreSQL**.
There is no Redis here, because this service has no job queue and no SSE pub/sub fan-out. It calls
the LLM backend directly and answers on the request.

| Object | Name |
|---|---|
| Deployment | `<release>-chatbot-api-chart` |
| Service | `<release>-chatbot-api-chart` (port 80 to container port `environment.PORT`) |
| ServiceAccount | `<release>-chatbot-api-chart` (when `serviceAccount.create`) |
| HorizontalPodAutoscaler | `<release>-chatbot-api-chart` (when `autoscaling.enabled`) |
| Ingress | `<release>-chatbot-api-chart` (when `ingress.enabled`) |

The container name inside the pod is `{{ .Chart.Name }}`, i.e. `chatbot-api-chart`, so use
`-c chatbot-api-chart` with `kubectl exec`.

Every non-secret variable is rendered inline into the Deployment from `.Values.environment`. This
chart has no `env-vars-configmap.yaml`.

## Values

Values live in three files, matching the convention used across bouc.io charts. There is no plain
`values.yaml`.

| File | Purpose |
|---|---|
| `base.values.yaml` | Values that never change between environments |
| `lcl.values.yaml` | Local cluster overrides |
| `snbx.values.yaml` | Sandbox overrides |

> In the cluster, FluxCD supplies values from generated ConfigMaps via `valuesFrom:`, not from these
> files directly. They are the source the ConfigMaps are generated from.

### Image

`image.registry` and `image.repository` are separate values, joined by the `chatbot-api-chart.image`
helper. The registry half is what a relocating operator overrides, either per release or through
`global.imageRegistry`. A `repository` whose first path segment already looks like a host is used
verbatim and `registry` is ignored, which keeps older full-string values rendering correctly.

### Database credentials

No password appears anywhere in this chart. `postgresql.auth.existingSecret` names a Secret that
External Secrets Operator populates, and it serves both consumers: the bundled Postgres reads the
`password` key, and the app reads the assembled `DATABASE_URL` key with `secretKeyRef`. The value is
**required** when Postgres is enabled, and the helper fails the render with an explicit message
rather than producing a workload that cannot authenticate.

`postgresql.auth.username` and `postgresql.auth.database` must stay in sync with the DSN that the
ExternalSecret template assembles, since the two build the same connection string from opposite
ends.

### LLM backend

| Value | Meaning |
|---|---|
| `environment.OLLAMA_BASE_URL` | In-cluster LLM endpoint |
| `environment.OLLAMA_MODEL` | Model name to request |
| `environment.OLLAMA_THINK` | Whether to let the model emit reasoning before answering |
| `environment.OLLAMA_SYSTEM_PROMPT` | System prompt. The shipped default is deliberately blunt about not restating or listing alternatives, because small models otherwise loop |
| `environment.MAX_HISTORY_MESSAGES` | How many prior turns are replayed as context |

### Memory retrieval

| Value | Default | Meaning |
|---|---|---|
| `environment.MEMORY_SERVICE_ENABLED` | `"false"` in base | Master switch for retrieval |
| `environment.MEMORY_SERVICE_URL` | empty in base | `memory-api-server` endpoint |
| `environment.MEMORY_SEARCH_LIMIT` | `"10"` | Maximum memories injected per turn |

### Post-conversation distillation

Distillation hands a finished conversation to `memory-distiller`, which decides what is worth
remembering. It is off in `base.values.yaml` and enabled per environment.

| Value | Default | Meaning |
|---|---|---|
| `environment.DISTILLATION_ENABLED` | `"false"` | Master switch |
| `environment.DISTILLATION_SERVICE_URL` | empty | Distiller endpoint |
| `environment.DISTILLATION_TIMEOUT_MS` | `"120000"` | Per-call timeout. Generous because the pipeline is several LLM calls deep |
| `environment.DISTILLATION_MIN_MESSAGES` | `"10"` | Shortest conversation worth distilling |
| `environment.DISTILLATION_MAX_MESSAGES` | `"50"` | Cap on what is sent in one pass |
| `environment.DISTILLATION_MIN_INTERVAL_MS` | `"900000"` | Cooldown between distillations of the same conversation |

The local values lower the thresholds substantially so the pipeline actually fires during
development.

### Annotations that do real work

- `deploymentAnnotations` carries `reloader.stakater.com/auto: "true"`, applied to the Deployment
  rather than the pod template. A `secretKeyRef` env var is resolved once at pod start and never
  refreshed, so without a restart a rotated database password never takes effect. Inert if Reloader
  is not installed.
- `podAnnotations` carries the Prometheus scrape trio (`prometheus.io/scrape`, `path` `/metrics`,
  `port`). The port value must match the container listen port or app-level metrics are silently
  never collected.

## Probes

The Deployment template renders `livenessProbe` and `readinessProbe` only when those values are set,
and **none of the three shipped values files set them**. Pods are therefore considered ready as soon
as the container starts. Supply the probes per environment if you need real health gating.

## Local usage

```bash
helm dependency build
helm lint . -f base.values.yaml -f lcl.values.yaml
helm template test . -f base.values.yaml -f lcl.values.yaml
helm install chatbot-api . -f base.values.yaml -f lcl.values.yaml
```

The values files layer: `base` first, then exactly one environment file. Pass them to `helm lint`
too: a bare `helm lint .` fails on a nil pointer, because there is no `values.yaml` for it to fall
back on.
`templates/tests/test-connection.yaml` provides a `helm test` connectivity check.

> The chart must be published to the chart registry by CI before FluxCD can reconcile it. Pushing
> chart source to git is not enough.

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
