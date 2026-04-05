# zoo

A GitOps homelab using two k3d clusters managed by ArgoCD over a Tailscale network.

## Clusters

| Name | Role | API port | k3d config |
|------|------|----------|------------|
| `k3d-mgmt` | Management — runs ArgoCD, manages both clusters | 6443 | `clusters/00-k3d-mgmt.yaml` |
| `k3d-apps` | Homelab — runs workloads | 6444 | `clusters/01-k3d-homelab.yaml` |

ArgoCD on `mgmt` reaches the `homelab` cluster via Tailscale (`https://k3d-apps.tail831c5d.ts.net`). The ArgoCD controller runs a Tailscale sidecar so it is on the Tailscale network without requiring a node-level agent.

## Repository layout

```
bootstrap/                        # Applied once manually to stand up core infrastructure
├── argocd/                       # ArgoCD — kustomize + Helm (argo-cd chart)
│   └── resources/
│       ├── values.yaml           # Helm values: Tailscale sidecar, cluster credentials
│       ├── ingress.yaml          # Tailscale ingress for the ArgoCD UI (hostname: argocd)
│       ├── project-default.yaml  # ArgoCD default AppProject
│       └── project-secrets.yaml  # ArgoCD secrets AppProject
├── sealed-secrets/               # Sealed Secrets controller — kustomize + Helm (kube-system)
└── tailscale-operator/           # Tailscale Kubernetes operator — per-cluster
    ├── mgmt/
    │   ├── kustomization.yaml
    │   ├── sealedsecret-mgmt.yaml   # Encrypted OAuth credentials (safe to commit)
    │   └── values-mgmt.yaml         # Helm values for mgmt operator
    └── homelab/
        ├── kustomization.yaml
        ├── sealedsecret-homelab.yaml
        └── values-homelab.yaml

gitops/                           # ArgoCD manages everything here after bootstrap
├── root-mgmt.yaml                # Root app-of-apps → gitops/mgmt/
├── root-homelab.yaml             # Root app-of-apps → gitops/homelab/
├── common/                       # Future: ApplicationSets targeting both clusters
├── mgmt/                         # Applications targeting the mgmt cluster
│   ├── sealed-secrets.yaml       # sync-wave -2
│   ├── tailscale-secrets.yaml    # sync-wave -1
│   └── tailscale-operator.yaml   # sync-wave  0
└── homelab/                      # Applications targeting the homelab cluster
    ├── sealed-secrets.yaml
    ├── tailscale-secrets.yaml
    └── tailscale-operator.yaml

clusters/                         # k3d cluster definitions
```

## Sync wave order

Within each `gitops/{mgmt,homelab}/` directory ArgoCD applies resources in waves:

| Wave | What |
|------|------|
| `-2` | Sealed Secrets controller (kube-system) |
| `-1` | Tailscale OAuth SealedSecret |
| `0`  | Tailscale Kubernetes operator |

## Bootstrap

### Prerequisites

- `k3d`, `kubectl`, `helm`, `kubeseal`, `kustomize`, `task` installed
- Tailscale OAuth client credentials (see below)

### Tailscale OAuth client setup

Create an OAuth client at [tailscale.com/admin/settings/oauth](https://tailscale.com/admin/settings/oauth) with the following scopes:

| Scope | Permission |
|-------|-----------|
| Devices Core | Write |
| Auth Keys | Write |
| Services | Write |

Add all three tags to your tailnet ACL **before** creating the OAuth client (tag permissions are baked in at client creation time):

```json
"tagOwners": {
    "tag:k8s-operator": [],
    "tag:kubernetes":   [],
    "tag:k8s":          [],
},
```

Also add all three tags to the OAuth client's **Tags** field in the console.

> **Notes:**
> - The operator requires OAuth client credentials (`tskey-client-...`), not a personal API key (`tskey-api-...`)
> - `tagOwners` must use empty `[]` — user-owned tags (`["autogroup:admin"]`) block OAuth client key creation
> - Create the ACL tags **first**, then create the OAuth client — tag grants are fixed at client creation time

The client ID and secret are stored in a `.env` file at the repo root (excluded from git):

```sh
TAILSCALE_CLIENT_ID=<oauth-client-id>
TAILSCALE_CLIENT_SECRET=<tskey-client-...>
```

The Taskfile loads `.env` automatically via `dotenv`, so no manual `export` is needed.

### First-time setup

```sh
# create .env with your credentials (see above), then:
task bootstrap
```

If clusters already exist from a previous run, use `task bootstrap:fresh` to tear them down and start clean.

This runs the following steps in order:

1. `cluster:create` — create both k3d clusters and fix kubeconfig ports
2. `bootstrap:sealed-secrets` — install Sealed Secrets controller on both clusters and wait for it to be ready
3. `secrets:seal:tailscale` — seal the Tailscale OAuth credentials with each cluster's public key
4. `bootstrap:tailscale` — add the Tailscale helm repo and install the operator on both clusters
5. `bootstrap:argocd` — install ArgoCD on `mgmt` (two-pass apply to handle CRD ordering)
6. `bootstrap:argocd:wait` — wait for ArgoCD server, repo-server, and applicationset-controller to be ready
7. `bootstrap:argocd:register-homelab` — create `argocd-manager` service account on `homelab` and register the cluster with ArgoCD
8. `bootstrap:root-apps` — apply `gitops/root-mgmt.yaml` and `gitops/root-homelab.yaml`
9. `bootstrap:argocd:password` — print the initial admin password

After step 8, ArgoCD self-manages everything under `gitops/` from this repo.

### Re-sealing credentials (e.g. after rotating the OAuth client)

Update `.env` with the new credentials, then:

```sh
task secrets:seal:tailscale
git add bootstrap/tailscale-operator/mgmt/sealedsecret-mgmt.yaml \
        bootstrap/tailscale-operator/homelab/sealedsecret-homelab.yaml
git commit -m "Reseal Tailscale credentials"

kubectl --context k3d-mgmt apply --server-side --force-conflicts -k bootstrap/tailscale-operator/mgmt
kubectl --context k3d-mgmt delete secret operator-oauth -n tailscale
kubectl --context k3d-mgmt rollout restart deploy/operator -n tailscale

kubectl --context k3d-homelab apply --server-side --force-conflicts -k bootstrap/tailscale-operator/homelab
kubectl --context k3d-homelab delete secret operator-oauth -n tailscale
kubectl --context k3d-homelab rollout restart deploy/operator -n tailscale
```

> The `delete secret` step is required because sealed-secrets suppresses updates when the decrypted value matches the existing secret. Deleting forces a clean recreation.

### Teardown

```sh
task teardown        # delete both clusters
task bootstrap:fresh # teardown + full bootstrap in one step
```

## Tailscale

Tailscale network: `khr.tail13637.ts.net`

The ArgoCD UI is exposed at `https://argocd` (Tailscale MagicDNS) via the Tailscale ingress controller on the `mgmt` cluster.
