# Infrastructure architecture notes

Companion to the application repository's `docs/ARCHITECTURE.md`, which
covers the system design in depth. This document covers decisions specific
to *this* repository — the Kubernetes/GitOps layer.

## Why Kustomize base + overlays, not one manifest set per environment

A naive "copy the staging folder, rename it production" approach means
every shared setting (probe paths, container ports, volume mounts, security
context) has to be kept in sync by hand across two full copies. Kustomize's
base/overlay model keeps exactly one copy of everything that's genuinely
shared, and overlays only ever contain the deltas: replica counts, resource
limits, image tags, ingress hosts, and a handful of ConfigMap values
(`CORS_ORIGIN`, `COOKIE_DOMAIN`, `RATE_LIMIT_MAX`). A reviewer can see the
entire diff between environments by reading two small overlay directories,
rather than diffing two large near-identical manifest trees.

## Why Argo CD Applications target different branches, not the same branch with different paths

Both Applications point at the same repository and both are auto-synced,
but `application-staging.yaml` tracks `main` while
`application-production.yaml` tracks a separate `production` branch. This
is deliberate: it keeps "auto-sync is on" (satisfying the assignment
requirement) fully compatible with "production only changes on a deliberate
action" (a real-world necessity). The gate isn't in Argo CD's sync
policy — it's upstream, in which Git ref gets updated and when. The
application repo's CI/CD workflow enforces this: every push to `main`
updates `overlays/staging` on the `main` branch here; only a version tag
(`vX.Y.Z`) updates `overlays/production` on the `production` branch here.

## Why StatefulSets for MongoDB and Redis instead of Deployments

Both need stable network identity and a PersistentVolumeClaim that survives
pod rescheduling — a plain Deployment with a PVC would work for a single
replica today, but StatefulSet is the correct primitive for anything
stateful even at replica count 1, and costs nothing extra to use correctly
from the start. `serviceName` + a headless Service (`clusterIP: None`) gives
each a stable DNS name (`mongo.ai-task-platform.svc.cluster.local`,
`redis.ai-task-platform.svc.cluster.local`) that survives pod restarts,
which is what the backend/worker ConfigMaps' `MONGO_URI`/`REDIS_URL` values
depend on.

## Why the worker has an HPA but a plain resource request/limit isn't enough on its own

Resource requests/limits (present on every container) tell the scheduler
how to place pods and prevent one runaway pod from starving its neighbors —
they don't by themselves add or remove replicas. The
`HorizontalPodAutoscaler` in `base/worker-hpa.yaml` (and its production
override, `overlays/production/worker-hpa-patch.yaml`) is what actually
changes replica count in response to load. Because workers are stateless
Redis consumers, this scaling is always safe: `BRPOP` guarantees Redis
delivers each queued task id to exactly one blocked caller, so adding or
removing worker replicas never causes a task to be double-processed or
dropped mid-flight (see the app repo's architecture doc §2 for the
graceful-shutdown behavior that also matters here during scale-down).

## Why `readOnlyRootFilesystem: true` on every container

All three application images are verified stateless — the backend and
worker never write to the filesystem at runtime (confirmed by inspecting
their source: all persistent state goes to MongoDB/Redis, never local
disk), and the frontend is a static nginx image. Setting
`readOnlyRootFilesystem: true` means a compromised process inside any of
these containers cannot write a webshell, modify application code, or
persist anything to disk — the container's root filesystem is immutable at
runtime. The frontend's nginx still needs *some* writable paths (PID file,
proxy/fastcgi cache directories) even when readonly, which is why it gets
three small `emptyDir` volumes mounted at exactly those paths rather than a
writable root filesystem.

## Why `dropped ALL capabilities` + `allowPrivilegeEscalation: false` everywhere

Standard Kubernetes Pod Security Standards ("restricted" profile) baseline:
none of these three services need any Linux capability beyond the default
unprivileged set, and none need to escalate privileges after starting.
Explicitly dropping `ALL` and re-adding nothing is a strictly stronger
posture than the container runtime's default capability set, at zero
functional cost to this application.
