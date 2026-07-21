# Managing secrets

None of the `stringData` values committed in `base/backend-secret.yaml`,
`overlays/staging/backend-secret-patch.yaml`, or
`overlays/production/backend-secret-patch.yaml` are real — they are
placeholders that exist only so `kustomize build` succeeds without any
external state, and so the required keys are documented in one place. Two
supported ways to replace them with real values follow.

## Required keys

| Key                  | Used for                                             |
| --------------------- | ----------------------------------------------------- |
| `JWT_ACCESS_SECRET`   | Signs short-lived access tokens                       |
| `JWT_REFRESH_SECRET`  | Signs long-lived refresh tokens (must differ from the above) |

Generate strong values with:

```bash
openssl rand -hex 48
```

Generate **two different** values — one per key, per environment. Never
reuse a secret between staging and production, and never reuse the access
secret as the refresh secret.

## Option A — manual `kubectl` apply (simplest, good for a single-operator setup)

Create the real Secret directly in the cluster, out of band from Git. It
will not be overwritten by Argo CD sync as long as the field values aren't
also declared in the tracked manifest — but since our tracked manifest *does*
declare placeholder values, `selfHeal: true` will actively revert a manually
`kubectl edit`-ed secret back to the placeholder on the next sync.

The safe pattern with `selfHeal` enabled: apply your real secret **before**
the first sync, using `kubectl apply --server-side` with a local, gitignored
file that overrides the same key:

```bash
# staging-secrets.local.yaml — NEVER commit this file
cat <<EOF > staging-secrets.local.yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: ai-task-platform-staging
type: Opaque
stringData:
  JWT_ACCESS_SECRET: "$(openssl rand -hex 48)"
  JWT_REFRESH_SECRET: "$(openssl rand -hex 48)"
EOF

kubectl apply -f staging-secrets.local.yaml
rm staging-secrets.local.yaml
```

Because Argo CD's `selfHeal` compares against what's *in Git*, and the
placeholder values are what's in Git, this will still eventually get
reverted on the next sync cycle. For a setup that's actually safe with
`selfHeal: true`, use Option B instead — this option is only appropriate if
you additionally remove the `backend-secret` resource from Git entirely
(delete it from the relevant `kustomization.yaml`) and manage it purely
out-of-band.

## Option B — Sealed Secrets (recommended; safe with `selfHeal: true`)

[Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
lets you commit an **encrypted** Secret to Git — only the in-cluster
controller (holding the matching private key) can decrypt it. Argo CD then
manages the `SealedSecret` resource like any other tracked manifest, and
`selfHeal` works correctly because the encrypted value in Git is the "real"
desired state.

```bash
# One-time: install the controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml

# Install the matching `kubeseal` CLI locally, then seal a real secret:
kubectl create secret generic backend-secret \
  --namespace ai-task-platform-staging \
  --from-literal=JWT_ACCESS_SECRET="$(openssl rand -hex 48)" \
  --from-literal=JWT_REFRESH_SECRET="$(openssl rand -hex 48)" \
  --dry-run=client -o yaml | kubeseal --format yaml > overlays/staging/backend-sealed-secret.yaml
```

Then:
1. Replace the `backend-secret-patch.yaml` reference in
   `overlays/staging/kustomization.yaml` with the generated
   `backend-sealed-secret.yaml` (as a `resource`, not a `patch`).
2. Commit `backend-sealed-secret.yaml` — it's ciphertext, safe to commit.
3. Repeat for `overlays/production` with its own independently-generated
   secret values.

## What never happens, regardless of option chosen

- Real secret values are never committed as plaintext to this repository.
- Staging and production never share a secret value.
- `.gitignore` in this repo excludes `*.local.yaml` and `*-secrets.yaml` as a
  backstop against accidentally committing a locally-generated real secret
  file.
