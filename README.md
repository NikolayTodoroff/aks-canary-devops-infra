# aks-canary-devops-infra

A two-tier voting application deployed on Azure Kubernetes Service using a canary deployment strategy, Azure DevOps Pipelines, and Helm for application packaging.

---

## Highlights

- AKS cluster with system and user node pools, Azure CNI Overlay networking, and Azure AD-integrated RBAC
- Canary deployment pattern implemented with three completely separate Helm releases — `voting-infra`, `voting-stable`, `voting-canary` — no conditionals, no shared resources across releases
- Kubernetes Network Policies restricting pod-level traffic: stable and canary pods accept inbound only from nginx, egress only to Redis and CoreDNS
- Readiness and liveness probes on both stable and canary tracks, ensuring pods only receive traffic once genuinely healthy
- Azure DevOps Pipelines with four dedicated pipelines — infrastructure, shared app infra, stable, and canary — each owning exactly one concern
- `Build.BuildId` as the image tag — unique per pipeline run, fully traceable, eliminates node-level image caching issues
- Horizontal Pod Autoscaler on the stable track only — canary replica count is always a deliberate human decision, never automated

---

## Repository Structure

```
aks-canary-devops-infra/
├── app/
│   └── azure-vote/
├── helm/
│   ├── voting-infra/                 
│   │   ├── Chart.yaml
│   │   ├── values.yml
│   │   ├── values-dev.yml
│   │   ├── values-prod.yml
│   │   └── templates/
│   │       ├── redis-deployment.yml
│   │       ├── redis-pvc.yml
│   │       ├── redis-service.yml
│   │       ├── ingress.yml
│   │       ├── role.yml
│   │       ├── rolebinding.yml
│   │       ├── clusterrole.yml
│   │       └── clusterrolebinding.yml
│   ├── voting-stable/                
│   │   ├── Chart.yaml
│   │   ├── values.yml
│   │   ├── values-dev.yml
│   │   ├── values-prod.yml
│   │   └── templates/
│   │       ├── deployment.yml
│   │       ├── service.yml
│   │       ├── hpa.yml
│   │       └── netpol.yml
│   └── voting-canary/                
│       ├── Chart.yaml
│       ├── values.yml
│       ├── values-dev.yml
│       ├── values-prod.yml
│       └── templates/
│           ├── deployment.yml
│           └── netpol.yml
├── infra/
│   ├── main/
│   ├── modules/
│   │   ├── aks/
│   │   ├── container-registry/
│   │   ├── networking/
│   │   └── monitoring/
│   └── env/
├── pipelines/
│   ├── infrastructure.yml            
│   ├── aks-infra.yml                 
│   ├── aks-stable.yml                
│   ├── aks-canary.yml                
│   ├── terraform.yml
│   └── helm-deploy.yml                  
│                 
├── scripts/
│   ├── bootstrap.sh
│   └── assign-roles.ps1
└── README.md
```

---

## Infrastructure

Both `dev` and `prod` environments provision identical resources:

| Resource | Name Pattern |
|---|---|
| Resource Group | `rg-main-akscanaryado-{env}` |
| Virtual Network | `vnet-akscanaryado-{env}` |
| AKS Node Subnet | `snet-aks-akscanaryado-{env}` |
| AKS Cluster | `aks-akscanaryado-{env}` |
| Container Registry | `acrakscanaryado{env}` |
| Log Analytics Workspace | `log-akscanaryado-{env}` |
| Action Group | `ag-akscanaryado-{env}` |

Two node pools per cluster: a system pool (`only_critical_addons_enabled = true`) running cluster-internal components, and a user pool running all application workloads. All pods use `nodeSelector: nodepool-type: user`.

---

## Canary Architecture

```
Internet
    → Azure Load Balancer (auto-provisioned, L4)
    → nginx Ingress Controller (L7, ingress-nginx Helm release)
    → voting-app Service (selector: app=voting-app, matches ALL pods)
    → voting-app-stable pods (track=stable, N replicas)
    → voting-app-canary pods  (track=canary, 1 replica)
    → Redis Service (ClusterIP)
    → Redis Pod → Azure Disk (PVC)
```

The traffic split is achieved purely through replica count — no service mesh, no weighted routing rules. With 4 stable replicas and 1 canary replica, approximately 80% of traffic hits stable and 20% hits canary. The `voting-app` Service selector matches `app: voting-app` only, deliberately omitting the `track` label so both Deployments receive traffic.

**Why three separate Helm releases:**
Each release owns exactly its own resources. `--wait` on `voting-stable` only watches stable pods. `--wait` on `voting-canary` only watches canary pods. No shared resources across releases means no Helm ownership conflicts, no orphaned resources from one track blocking the other, and no timeout caused by the other track's health state. Shared infrastructure (Redis, Ingress, RBAC) lives in `voting-infra`, deployed once and never touched during canary operations.

---

## Network Policies

Network Policies enforce pod-level traffic rules — the Kubernetes equivalent of NSGs but operating inside the cluster rather than at the VNet level.

**Stable and canary pods:**
- Inbound: accept only from the `ingress-nginx` namespace on TCP 80
- Outbound: allow only to Redis pods on TCP 6379, and to CoreDNS (`kube-system/kube-dns`) on UDP 53 and TCP 53

**Both UDP and TCP 53 are required** — DNS primarily uses UDP but falls back to TCP for large responses. Blocking TCP 53 causes intermittent, hard-to-debug DNS failures.

**`network_policy = "azure"` is required in the AKS cluster config** — without it, NetworkPolicy objects exist as Kubernetes API objects but the enforcement engine is absent and all policies are silently ignored.

---

## CI/CD Architecture

Four dedicated pipelines, each owning exactly one concern:

### `infrastructure.yml` — Terraform
```
Validate (fmt, init, validate, tflint)
    ↓
Plan (terraform plan → artifact)
    ↓
Apply (deployment job, infrastructure-{env} environment, approval gate on prod)
```

### `aks-infra.yml` — Shared cluster resources
```
SetupIngress (helm upgrade ingress-nginx)
    ↓
DeployVotingInfra (helm upgrade voting-infra)
```

### `aks-stable.yml` — Stable track
```
Build (docker build → push to ACR with Build.BuildId tag)
    ↓
DeployStable (helm upgrade voting-stable)
```

Triggers automatically on pushes to `app/**` and `helm/voting-stable/**`. Uses `Build.BuildId` as the image tag — unique per pipeline run.

### `aks-canary.yml` — Canary track
```
DeployCanary (helm upgrade voting-canary)
```

`trigger: none` — canary is always a deliberate manual decision. You supply the `Build.BuildId` from the validated stable run as the `imageTag` parameter, ensuring canary always runs a previously validated image.

### Reusable template — `templates/helm-deploy.yml`
Used by all three application pipelines. Accepts `stageName`, `displayName`, `environment`, `aksResourceGroup`, `aksClusterName`, and `helmCommand` as parameters. Handles `az aks get-credentials` and `kubelogin convert-kubeconfig -l azurecli` before running the helm command.

---

## Key Design Decisions

- **Three separate Helm releases, not one.** `voting-infra`, `voting-stable`, and `voting-canary` are completely independent releases with no shared resources. Each pipeline's `--wait` only watches its own release's resources. This eliminates the entire class of cross-track blocking issues that arise from mixing concerns in one release.

- **Four focused pipelines.** Infrastructure (Terraform), shared app infrastructure (ingress + voting-infra), stable track, and canary track are completely separate pipelines. Changing application code doesn't trigger infrastructure changes. Deploying canary doesn't rebuild the image or touch stable.

- **`network_policy = "azure"` in AKS cluster config.** Without this, NetworkPolicy objects are accepted by the API server but silently ignored — there is no enforcement engine. Adding it deploys Azure Network Policy Manager as a DaemonSet that actually enforces the rules.

- **DNS egress scoped precisely.** The NetworkPolicy DNS egress rule targets `kube-system` namespace AND `k8s-app: kube-dns` pod label simultaneously (AND logic, same list item). Both UDP 53 and TCP 53 are allowed — DNS falls back to TCP for large responses and blocking TCP 53 causes intermittent failures.

- **`Build.BuildId` as image tag.** ADO's predefined build counter is available in every stage without any setup. Unlike mutable tags (`:stable`, `:canary`), a numeric build ID never collides with a previously cached image on any node. To deploy the same image to prod that was validated on dev, note the dev `Build.BuildId` and supply it explicitly when queuing the prod pipeline.

- **HPA on stable only.** Canary replica count is always a deliberate human decision — you control exactly how much traffic hits the new version. An HPA on canary would autonomously increase canary exposure under load, defeating the purpose of controlled progressive delivery.

---

## Security

| Mechanism | Purpose |
|---|---|
| Workload Identity Federation | Pipeline authentication — no stored credentials |
| Kubelet managed identity + AcrPull | Credential-free image pulls from ACR |
| Azure Kubernetes Service RBAC Cluster Admin | Pipeline data-plane access to Kubernetes API |
| Azure Kubernetes Service Cluster Admin Role | Pipeline management-plane credential retrieval |
| Azure AD-integrated cluster RBAC | kubectl via AAD group membership, no static kubeconfig |
| `imagePullPolicy: Always` | Prevents stale cached images on nodes |
| Kubernetes Network Policies | Pod-level traffic restrictions, deny-by-default per pod |
| `network_policy = "azure"` | Enables enforcement engine for NetworkPolicy objects |

---

## Technologies

- **Terraform** — AKS cluster, ACR, networking, monitoring
- **Azure DevOps Pipelines** — four dedicated pipelines, reusable YAML templates, environment approval gates
- **Azure Boards** — Scrum template, Epic → Feature → PBI → Task hierarchy, AB#N commit linking
- **Helm** — three separate charts per concern, per-environment values, no cross-chart conditionals
- **Azure Kubernetes Service** — managed Kubernetes, system+user node pools, Azure CNI Overlay, AAD-integrated RBAC
- **Azure Container Registry** — per-environment registries, kubelet managed identity pulls
- **nginx Ingress Controller** — Layer 7 routing, dedicated cluster-level Helm release
- **Kubernetes Network Policies** — pod-level traffic enforcement, scoped DNS egress
- **Horizontal Pod Autoscaler** — CPU-based scaling on stable track only
- **Python/Flask** — voting app frontend
- **Redis** — vote persistence, PVC-backed Azure Disk
- **Azure Monitor** — Log Analytics, Container Insights, metric alerts
- **TFLint** — Terraform static analysis and security scanning

---
