# Production Deployment — Kubernetes (minikube / OpenShift)

Production-oriented orchestration for the **Secure Event Ticketing Platform**.
All objects live in the `ticketing` namespace.

## Services

| Service  | Kind        | Replicas | Port | Storage   | Exposed                     |
| -------- | ----------- | -------- | ---- | --------- | --------------------------- |
| postgres | StatefulSet | 1        | 5432 | PVC (1Gi) | internal (headless Service) |
| redis    | Deployment  | 1        | 6379 | none      | internal                    |
| api      | Deployment  | 2        | 8080 | none      | via Ingress `/api`          |
| frontend | Deployment  | 2        | 3000 | none      | via Ingress `/`             |
| worker   | Deployment  | 1        | —    | none      | none (queue consumer)       |

Config is split between a **ConfigMap** (`ticketing-config`, non-sensitive) and a
**Secret** (`ticketing-secret`, DB credentials). No credentials are hardcoded in
any manifest.

---

## Prerequisites

- `kubectl` (v1.34+)
- `minikube` (Docker driver) — add it to PATH if needed:
  `C:\Program Files\Kubernetes\Minikube`
- Docker (to build images)

```powershell
minikube start
minikube addons enable ingress      # nginx ingress controller
```

---

## 1. Build images

The Kubernetes manifests run the **`prod`** stage (distroless, non-root), not `dev`.

```powershell
docker build -t ticketing-api:1.0      --target prod ./api
docker build -t ticketing-frontend:1.0 --target prod ./frontend
docker build -t ticketing-worker:1.0   --target prod ./worker
```

### Make images visible to the cluster

**minikube (local)** — load the locally built images into the cluster:

```powershell
minikube image load ticketing-api:1.0
minikube image load ticketing-frontend:1.0
minikube image load ticketing-worker:1.0
```

The Deployments use `imagePullPolicy: IfNotPresent` so the cluster uses these
local images instead of trying to pull from a registry.

**Production (registry)** — CI publishes images to GHCR
(`ghcr.io/inkoc/ticketing-<service>`) tagged with the immutable commit SHA and
`latest`. For a real deployment, point the manifests at the **SHA tag**
(immutable; avoid `:latest` in production) and add an `imagePullSecret` if the
package is private:

```yaml
image: ghcr.io/inkoc/ticketing-api:<commit-sha>
```

---

## 2. Secret

`secret.yaml` is **git-ignored** (it holds the real password). Create it from the
committed template the first time:

```powershell
Copy-Item infra/k8s/secret.example.yaml infra/k8s/secret.yaml
# edit infra/k8s/secret.yaml and set a real POSTGRES_PASSWORD
```

---

## 3. Deploy

Order matters: namespace and config first, then the init ConfigMap, then everything else.

```powershell
kubectl apply -f infra/k8s/namespace.yaml
kubectl apply -f infra/k8s/configmap.yaml
kubectl apply -f infra/k8s/secret.yaml

# DB schema, generated from the infra/postgres/init.sql
kubectl create configmap postgres-init `
  --from-file=init.sql=infra/postgres/init.sql `
  -n ticketing

kubectl apply -f infra/k8s/
```

Watch it come up:

```powershell
kubectl get pods -n ticketing -w
```

All pods should reach `Running` / `READY 1/1`. The `worker` may restart a few
times on first boot until Postgres is reachable — this is expected.

---

## 4. External access (Ingress)

The Ingress routes one host to two backends:

- `http://ticketing.local/` → frontend (3000)
- `http://ticketing.local/api` → api (8080)

Map the host name to the cluster. On minikube with the Docker driver run a tunnel
(keep it open in a separate terminal):

```powershell
minikube tunnel
```

Then add to `C:\Windows\System32\drivers\etc\hosts`:

```
127.0.0.1   ticketing.local
```

(Or use `minikube ip` and map that address instead of `127.0.0.1`.)

---

## 5. Validate

```powershell
# health + readiness (readiness checks Postgres + Redis)
kubectl exec -n ticketing deploy/api -- /nodejs/bin/node -e "require('http').get('http://localhost:8080/readyz',r=>{let d='';r.on('data',c=>d+=c);r.on('end',()=>console.log(d))})"

# full workflow via port-forward (purchase -> queue -> worker -> postgres -> read back)
Start-Process kubectl -ArgumentList "port-forward -n ticketing svc/api 8080:8080"
Invoke-RestMethod -Method Post -Uri http://localhost:8080/tickets/purchase `
  -ContentType 'application/json' `
  -Body '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
Invoke-RestMethod -Uri http://localhost:8080/tickets/orders   # status should be "processed"
```

---

## 6. Rolling update & rollback

```powershell
# build + load a new tag, then roll it out one replica at a time (zero downtime)
docker tag ticketing-api:1.0 ticketing-api:1.1
minikube image load ticketing-api:1.1
kubectl set image deployment/api api=ticketing-api:1.1 -n ticketing
kubectl annotate deployment/api -n ticketing kubernetes.io/change-cause="upgrade to 1.1" --overwrite
kubectl rollout status deployment/api -n ticketing

# inspect history and roll back
kubectl rollout history deployment/api -n ticketing
kubectl rollout undo deployment/api -n ticketing
kubectl rollout status deployment/api -n ticketing
```

---

## Security notes

- **Secrets / config**: credentials only in the Secret; non-sensitive values in
  the ConfigMap.
- **Least privilege**: each workload has its own ServiceAccount with
  `automountServiceAccountToken: false` (no API access — none of these apps call
  the Kubernetes API). App containers run with `readOnlyRootFilesystem`,
  `allowPrivilegeEscalation: false`, and all capabilities dropped, on top of
  distroless non-root images.
- **Network segmentation**: `networkpolicy.yaml` is default-deny ingress plus
  explicit allows (controller→web, api/worker→postgres, api/worker→redis).
  **Note:** minikube's default CNI does not _enforce_ NetworkPolicy — start with
  `minikube start --cni=calico` to enforce it. The policies encode the intended
  segmentation regardless.
- **Image scanning**: performed in CI (Trivy, HIGH/CRITICAL, build fails on
  fixable findings). The report is published as the `trivy-<service>-report` CI
  artifact.

---

## Incident runbook

Quick triage first:

```powershell
kubectl get pods -n ticketing
kubectl describe pod <pod> -n ticketing      # see Events at the bottom
kubectl logs <pod> -n ticketing [--previous] # --previous = last crashed container
```

### 1. Database down / not ready

**Symptoms:** `api` `/readyz` returns 503; `worker` `CrashLoopBackOff`; api pods not READY.
**Diagnose:**

```powershell
kubectl get pods -n ticketing -l app=postgres
kubectl logs postgres-0 -n ticketing
kubectl describe pod postgres-0 -n ticketing
```

**Resolve:** wait for `postgres-0` to become READY (api/worker recover
automatically once it is). If the PVC is corrupt and there's no data to keep:

```powershell
kubectl delete statefulset postgres -n ticketing
kubectl delete pvc data-postgres-0 -n ticketing
kubectl apply -f infra/k8s/postgres.yaml
```

### 2. Bad image tag

**Symptoms:** pod stuck `ImagePullBackOff` / `ErrImagePull`, or `ErrImageNeverPull`.
**Diagnose:** `kubectl describe pod <pod> -n ticketing` → see the image name/tag it
can't get.
**Resolve:** roll back to a known-good revision, or fix the tag:

```powershell
kubectl rollout undo deployment/<name> -n ticketing
# or
kubectl set image deployment/<name> <container>=<good-image:tag> -n ticketing
```

On minikube, `ErrImageNeverPull` usually means the image wasn't loaded — run
`minikube image load <image:tag>`.

### 3. Broken / missing secret

**Symptoms:** pod `CreateContainerConfigError`, or Postgres auth failures in logs.
**Diagnose:**

```powershell
kubectl describe pod <pod> -n ticketing       # "secret ... not found" / "key ... not found"
kubectl get secret ticketing-secret -n ticketing -o jsonpath='{.data}'
```

**Resolve:** (re)create the Secret with the right keys, then restart consumers:

```powershell
kubectl apply -f infra/k8s/secret.yaml
kubectl rollout restart deployment/api deployment/worker -n ticketing
kubectl delete pod postgres-0 -n ticketing
```

The Secret keys must be exactly `POSTGRES_USER` and `POSTGRES_PASSWORD`, and must
match what Postgres was first initialized with (credentials are baked in at first
init — changing them later requires reinitializing the DB volume).

### 4. Worker CrashLoopBackOff on startup (ordering)

**Symptoms:** `worker` logs `getaddrinfo ENOTFOUND postgres` or connection refused
shortly after deploy.
**Cause:** worker started before Postgres/Redis were ready; it exits and Kubernetes
restarts it with backoff.
**Resolve:** usually self-heals once Postgres is READY. To clear the backoff
immediately: `kubectl delete pod -l app=worker -n ticketing`.

### 5. Ingress not reachable from host

**Symptoms:** `ticketing.local` doesn't resolve or connection refused.
**Diagnose:**

```powershell
kubectl get ingress -n ticketing
kubectl get pods -n ingress-nginx
```

**Resolve:** ensure `minikube addons enable ingress`, keep `minikube tunnel`
running, and confirm the hosts-file entry. As a fallback, use
`kubectl port-forward -n ticketing svc/frontend 3000:3000`.
