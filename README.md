# oldest-first-scaler

A Helm chart that ensures the **oldest pod is terminated first** during scale-in events, by automatically managing [`pod-deletion-cost`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#pod-deletion-cost) annotations.

## How it works

A lightweight CronJob runs on a configurable schedule and:

1. Lists all **Running** pods in the release namespace that match the configured label selector.
2. Sorts them by `creationTimestamp` (oldest → newest).
3. Assigns `controller.kubernetes.io/pod-deletion-cost` starting at `startCost` (default `0`), incrementing by `costIncrement` (default `100`) for each subsequent pod.

Kubernetes uses `pod-deletion-cost` as a hint when choosing which pod to remove during scale-in: the pod with the **lowest** cost is preferred — so the oldest pod is always evicted first.

Works with standard `Deployment` / `ReplicaSet` scale-in and [Argo Rollouts](https://argoproj.github.io/rollouts/).

## Prerequisites

- Kubernetes 1.22+
- Helm 3.2+

## Installation

```bash
helm repo add oms1226 https://oms1226.github.io/oldest-first-scaler
helm repo update

helm install oldest-first-scaler oms1226/oldest-first-scaler \
  --namespace <target-namespace> \
  --set podSelector.labels.app=my-app
```

## Uninstallation

```bash
helm uninstall oldest-first-scaler --namespace <target-namespace>
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `podSelector.labels` | Key/value label selector for target pods | `{}` (all pods) |
| `schedule` | CronJob schedule (cron syntax) | `*/30 * * * *` |
| `startCost` | Deletion cost assigned to the oldest pod | `0` |
| `costIncrement` | Cost increment per subsequent pod | `100` |
| `serviceAccount.create` | Create a dedicated ServiceAccount | `true` |
| `serviceAccount.name` | ServiceAccount name override | `""` (auto) |
| `serviceAccount.annotations` | Annotations for the ServiceAccount | `{}` |
| `image.repository` | Container image for the updater | `bitnami/kubectl` |
| `image.tag` | Image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `imagePullSecrets` | Image pull secrets | `[]` |
| `resources.requests.cpu` | CPU request | `10m` |
| `resources.requests.memory` | Memory request | `32Mi` |
| `resources.limits.cpu` | CPU limit | `100m` |
| `resources.limits.memory` | Memory limit | `64Mi` |
| `successfulJobsHistoryLimit` | Number of successful job runs to retain | `1` |
| `failedJobsHistoryLimit` | Number of failed job runs to retain | `1` |
| `podAnnotations` | Extra annotations for the CronJob pods | `{}` |
| `nodeSelector` | Node selector for CronJob pods | `{}` |
| `tolerations` | Tolerations for CronJob pods | `[]` |
| `affinity` | Affinity rules for CronJob pods | `{}` |
| `nameOverride` | Override the chart name | `""` |
| `fullnameOverride` | Override the full resource name | `""` |

### Example: target a specific Deployment

```yaml
# values-my-app.yaml
podSelector:
  labels:
    app: my-app
    environment: production

schedule: "*/15 * * * *"   # every 15 minutes
costIncrement: 100
```

```bash
helm install oldest-first-scaler oms1226/oldest-first-scaler \
  --namespace my-namespace \
  -f values-my-app.yaml
```

### Example: Argo Rollouts

```yaml
podSelector:
  labels:
    app: edge-grpc
    rollouts-pod-template-hash: ""   # matches any value — key presence only
```

> **Note**: Kubernetes label selectors require a value. To match on key presence
> only, use the full selector syntax via `kubectl` directly or pin to a known
> hash. The chart passes labels as `key=value` pairs; set the value to the
> expected hash string if using Argo Rollouts.

## RBAC

The chart creates a `Role` with `get`, `list`, and `patch` permissions on `pods`
in the release namespace only. No cluster-wide permissions are needed.

## License

Apache 2.0 — see [LICENSE](LICENSE).
