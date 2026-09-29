# Kubernetes Microservices Demo

Kustomize-managed manifests for a small production-style microservices stack: an
HTTP **api** service and a background **worker** service. Built to demonstrate
the habits that matter in real clusters — health probes, resource budgets,
hardened pod security, autoscaling, disruption budgets, and default-deny
network policy — with environment overlays for dev and prod.

> **Note:** the container images here are small public images (`hashicorp/http-echo`,
> `alpine`) so the stack deploys and runs as-is. In a real pipeline you would
> point `api-deployment.yaml` / `worker-deployment.yaml` at images built by
> your CI (e.g. `ghcr.io/<org>/api:<sha>`).

## Architecture

Everything lives in a single namespace, `microservices`.

**Topology**

- **Ingress** (`api-ingress`, NGINX ingress class, TLS via cert-manager) is the
  only entry point from outside the cluster. It terminates TLS and routes
  `https://<host>/` to the `api` ClusterIP Service.
- **api** is a stateless HTTP Deployment behind a ClusterIP Service. It is
  scaled by a HorizontalPodAutoscaler (CPU), protected by a
  PodDisruptionBudget, and rolls with `maxUnavailable: 0` for zero-downtime
  deploys. Liveness and readiness probes gate traffic.
- **worker** is a background Deployment with no Service — it is not reachable
  from outside. It calls the `api` Service internally over the cluster network
  (e.g. to process jobs) on port 8080.
- **Config** comes from a ConfigMap (`app-config`); **credentials** come from a
  Secret (`app-secrets`). The checked-in Secret is a template — every value is
  `replace-me` and must be filled in before deploying.
- **NetworkPolicy** defaults to deny-all (ingress *and* egress). Explicit
  policies then allow exactly three things: ingress-controller → api:8080,
  worker → api:8080, and DNS egress to kube-system. Nothing else can talk to
  anything.
- Both workloads run as non-root with a read-only root filesystem, no
  privilege escalation, all capabilities dropped, and a RuntimeDefault seccomp
  profile. Neither mounts the ServiceAccount token.

**Traffic flow**

1. Client → `https://api.example.com/` (prod) → NGINX Ingress Controller
2. Ingress → `api:80` (ClusterIP) → one of the api pods `:8080`
   (only pods passing the readiness probe receive traffic)
3. On a schedule, `worker` → `http://api:8080/` internally to do its work
4. HPA adds api replicas as CPU crosses 70%; PDB guarantees `minAvailable`
   pods stay up during voluntary disruptions (node drains, upgrades)

## Prerequisites

- `kubectl` ≥ 1.27 (ships with `kubectl kustomize` built in), **or**
- standalone [`kustomize`](https://kubectl.docs.kubernetes.io/installation/kustomize/) ≥ 5
- A cluster with an NGINX ingress controller and cert-manager installed
  (only needed for the Ingress/TLS pieces; everything else applies anywhere)

## Deploy

```bash
# 1. Fill in every replace-me value in base/secret.yaml
# 2. Deploy an overlay:
kubectl apply -k overlays/dev     # 1 api replica, light resources, api.dev.example.com
kubectl apply -k overlays/prod    # 3 api replicas, larger resources, api.example.com

# 3. Verify
kubectl -n microservices get pods,svc,ingress,hpa,pdb
kubectl -n microservices port-forward svc/api 8080:80
curl localhost:8080   # -> api-ok
```

To tear down: `kubectl delete -k overlays/<env>`.

## Repo structure

```
.
├── base/                      # Environment-agnostic manifests
│   ├── namespace.yaml         # microservices namespace
│   ├── serviceaccount.yaml    # app-sa (token automount disabled)
│   ├── configmap.yaml         # non-sensitive config (log level, intervals)
│   ├── secret.yaml            # Secret TEMPLATE — replace-me values only
│   ├── api-deployment.yaml    # api: probes, resources, hardened securityContext
│   ├── api-service.yaml       # ClusterIP Service for api
│   ├── worker-deployment.yaml # worker: no Service, calls api internally
│   ├── hpa.yaml               # CPU-based autoscaling for api
│   ├── pdb.yaml               # disruption budget for api
│   ├── ingress.yaml           # NGINX ingress + cert-manager TLS
│   ├── networkpolicy.yaml     # default-deny + explicit allows
│   └── kustomization.yaml
├── overlays/
│   ├── dev/kustomization.yaml  # 1 replica, small resources, dev host, debug logs
│   └── prod/kustomization.yaml # 3 replicas, larger resources, prod host, stricter PDB
├── .github/workflows/validate.yml  # CI: kustomize build on base + both overlays
└── README.md
```

### Configuration

| Key | Source | Used by | Default |
|---|---|---|---|
| `LOG_LEVEL` | ConfigMap | api, worker | `info` (`debug` in dev) |
| `WORKER_INTERVAL_SECONDS` | ConfigMap | worker | `30` |
| `DATABASE_URL` | Secret | api | `replace-me` |
| `API_KEY` | Secret | api, worker | `replace-me` |

## What I'd add next

- **external-secrets** (backed by AWS Secrets Manager / Vault) instead of a
  checked-in Secret template — the `replace-me` Secret exists only so this repo
  is deployable as a demo without external dependencies.
- **cert-manager** `ClusterIssuer` manifests (the Ingress is already annotated;
  the issuer itself would live in a platform layer outside this app repo).
- **Pod anti-affinity / topologySpreadConstraints** on the api Deployment so
  prod replicas spread across zones.
- **ResourceQuota + LimitRange** on the namespace to bound noisy neighbors.
- **ServiceMonitor / PodMonitor** for Prometheus scraping, plus alerts on HPA
  saturation and elevated 5xx rate.
- **Private registry + imagePullSecrets**, with image digests pinned by CI.
- **GitOps** (Argo CD Application per overlay) so deploys are pull-based.
- **Policy enforcement** (Kyverno/OPA Gatekeeper) to require the security
  posture shown here on every workload in the cluster.
