# Bootstrap: installing Argo CD on a fresh k3s / Kubernetes cluster

This installs Argo CD itself. It only needs to be done once per cluster —
after this, everything else (the app, its namespaces, its scaling) is
managed declaratively through the `Application` manifests in `../argocd/`.

## Prerequisites

- A running Kubernetes cluster (k3s is sufficient — see the root README for
  a quick k3s install if you don't have one yet)
- `kubectl` configured to point at that cluster
- (Recommended) `argocd` CLI installed locally for convenience:
  https://argo-cd.readthedocs.io/en/stable/cli_installation/

## 1. Create the `argocd` namespace and install Argo CD

```bash
kubectl apply -f bootstrap/argocd-namespace.yaml

# Official install manifest, pinned to a known-good version:
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.4/manifests/install.yaml

# Wait for all Argo CD components to become ready
kubectl -n argocd wait --for=condition=available --timeout=300s deployment --all
```

## 2. Access the Argo CD UI

```bash
# Port-forward the API server/UI locally
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Open https://localhost:8080 — accept the self-signed cert warning.

Retrieve the initial admin password (Argo CD auto-generates one on first
install, stored in a Secret):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

Log in as `admin` with that password. **Change it immediately** after first
login (User Info → Update Password), then delete the bootstrap secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## 3. Register this repository with Argo CD

Argo CD needs read access to this Git repository to sync from it. For a
public repo this step is optional; for a private repo, register it (via UI
`Settings → Repositories → Connect Repo`, or CLI):

```bash
argocd login localhost:8080
argocd repo add https://github.com/<your-org>/waygood-ai-task-platform-infra.git \
  --username <git-username> \
  --password <personal-access-token>
```

## 4. Create the AppProject and both Applications

```bash
kubectl apply -f argocd/appproject.yaml
kubectl apply -f argocd/application-staging.yaml
kubectl apply -f argocd/application-production.yaml
```

Argo CD will immediately start syncing:
- `overlays/staging` → namespace `ai-task-platform-staging`, tracking the
  `main` branch
- `overlays/production` → namespace `ai-task-platform-prod`, tracking the
  `production` branch

## 5. Populate real secrets before the first sync completes successfully

The committed `backend-secret` manifests contain placeholder values — see
[`../docs/SECRETS.md`](../docs/SECRETS.md) before relying on either
environment. Argo CD will happily sync the placeholders (it doesn't know
they're fake), so the backend pods will crash-loop authenticating JWTs with
a secret everyone can read in this public repo until you replace them.

## 6. Verify

```bash
kubectl get applications -n argocd
argocd app get ai-task-platform-staging
```

Take the dashboard screenshot for the assignment submission from the Argo CD
UI's main tree/list view once both Applications show `Synced` / `Healthy`.
