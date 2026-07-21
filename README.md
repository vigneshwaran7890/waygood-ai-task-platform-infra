# AI Task Platform — Infrastructure Repository

Kubernetes manifests and GitOps configuration for the
[AI Task Processing Platform](https://github.com/vigneshwaran7890/waygood-ai-task-platform),
managed with Kustomize and deployed via Argo CD.

This repository is the single source of truth for what runs in each
environment — nothing is deployed by running `kubectl apply` by hand;
Argo CD continuously reconciles the cluster to match what's committed here.

## Repository layout

```
.
├── base/                    # Environment-agnostic manifests, the shared foundation
│   ├── namespace.yaml
│   ├── backend-deployment.yaml / backend-service.yaml / backend-configmap.yaml / backend-secret.yaml
│   ├── frontend-deployment.yaml / frontend-service.yaml
│   ├── worker-deployment.yaml / worker-hpa.yaml / worker-configmap.yaml
│   ├── mongo-statefulset.yaml
│   ├── redis-statefulset.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── staging/              # 1 replica each, relaxed limits, staging.<domain>
│   └── production/           # 3+ replicas, HPAs on backend+worker, <domain>
├── argocd/
│   ├── appproject.yaml               # scopes what Applications in this project may touch
│   ├── application-staging.yaml      # tracks `main`, namespace ai-task-platform-staging
│   └── application-production.yaml   # tracks `production` branch, namespace ai-task-platform-prod
├── bootstrap/                # One-time Argo CD install on a fresh cluster
└── docs/
    ├── SECRETS.md             # how real secrets are provisioned (never committed)
    └── ARCHITECTURE-NOTES.md  # infra-specific decisions and rationale
```

## Environments

| Environment | Namespace                    | Git ref tracked | Replicas (backend / frontend / worker) | Host |
| ----------- | ------------------------------ | ----------------- | ---------------------------------------- | ---- |
| Staging     | `ai-task-platform-staging`     | `main`             | 1 / 1 / 1 (HPA 1–3 on worker)             | `staging.ai-task-platform.example.com` |
| Production  | `ai-task-platform-prod`        | `production`       | 3 / 3 / 3 (HPA 3–10 backend, 3–15 worker) | `ai-task-platform.example.com` |

Promotion from staging to production is a deliberate Git action — merging
(or fast-forwarding) `main` into the `production` branch — not something
that happens automatically on every commit. See the CI/CD workflow in the
application repository (`.github/workflows/ci-cd.yml`) for exactly how each
branch/tag maps to an update here.

## Quick start (fresh k3s cluster, from zero)

### 1. Install k3s (skip if you already have a cluster)

```bash
curl -sfL https://get.k3s.io | sh -
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
# Adjust the server URL in ~/.kube/config if running kubectl from outside the node
```

k3s ships with Traefik as its Ingress controller by default, which is what
`base/ingress.yaml` targets (`ingressClassName: traefik`) — no extra install
needed for that piece.

### 2. Install Argo CD

Follow [`bootstrap/README.md`](bootstrap/README.md) in full. Summary:

```bash
kubectl apply -f bootstrap/argocd-namespace.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.4/manifests/install.yaml
kubectl -n argocd wait --for=condition=available --timeout=300s deployment --all
```

### 3. Provision real secrets

Follow [`docs/SECRETS.md`](docs/SECRETS.md) — do this **before** step 4, or
the backend pods will crash-loop with placeholder JWT secrets that are
public in this repo's Git history.

### 4. Register the Argo CD Applications

```bash
kubectl apply -f argocd/appproject.yaml
kubectl apply -f argocd/application-staging.yaml
kubectl apply -f argocd/application-production.yaml
```

Argo CD takes it from there — auto-sync (`prune: true`, `selfHeal: true`) is
enabled on both, so the cluster converges to match this repo within seconds
of any change, and any manual `kubectl edit` drift is reverted automatically.

### 5. Verify

```bash
kubectl get applications -n argocd
kubectl get pods -n ai-task-platform-staging
kubectl get pods -n ai-task-platform-prod
```

Both Applications should show `Synced` and, once pods pass their readiness
probes, `Healthy`.

## Making a change

Everything here is built with `kubectl kustomize` (no cluster access
needed to validate a change):

```bash
kubectl kustomize overlays/staging    # render the full staging manifest set
kubectl kustomize overlays/production
```

Common changes:

- **Bump an image tag manually** (normally done by CI, but for a manual
  rollback): `cd overlays/staging && kustomize edit set image docker.io/vignesh0756/ai-task-backend=docker.io/vignesh0756/ai-task-backend:<tag>`
- **Change replica count / resource limits**: edit the relevant
  `overlays/<env>/*-patch.yaml` file — never edit `base/*` for an
  environment-specific tweak; `base/` is the shared foundation both
  overlays patch from.
- **Add a new Kubernetes resource**: add the manifest under `base/`, then
  reference it from `base/kustomization.yaml`'s `resources:` list.

Commit and push to `main` (staging) or fast-forward `production` (see the
CI/CD workflow) — Argo CD picks it up automatically.

## Design notes

- **Kustomize over Helm**: no templating language to learn to review a
  diff; overlays are plain strategic-merge patches against the shared
  `base/`, which is enough for two environments with different scale and a
  handful of differing config values.
- **In-cluster MongoDB/Redis**: `base/mongo-statefulset.yaml` and
  `base/redis-statefulset.yaml` run both as StatefulSets with
  PersistentVolumeClaims, so the entire stack — including its data layer —
  deploys from this repo alone on a bare cluster, with no external managed
  service dependency. Redis runs with AOF persistence (`appendonly yes`) so
  queued task ids survive a pod restart. For a larger production footprint,
  swapping these for a managed MongoDB Atlas cluster / managed Redis is a
  drop-in change to `backend-configmap.yaml`'s `MONGO_URI`/`REDIS_URL` —
  nothing else in the app needs to change.
- **Worker horizontal scaling**: `base/worker-hpa.yaml` scales on CPU
  utilization. Workers are stateless Redis consumers (`BRPOP` guarantees
  each queued task id is delivered to exactly one replica), so scaling this
  number up or down is always safe — no coordination logic needed.
- **Traefik, not nginx-ingress**: matches k3s's built-in ingress controller,
  so there's nothing extra to install on the simplest supported cluster.
