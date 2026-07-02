# homelabinfra-k8s-apps

Kubernetes application manifests for the NNT homelab, deployed via Helm and ArgoCD. This repo is the GitOps source of truth — ArgoCD watches it continuously and reconciles the cluster to match what's here.

Pushing a change to this repo is a deployment. ArgoCD handles the diff, apply, and retry. No manual `kubectl apply`, no pipeline trigger needed.

---

## Architecture

```
Git push (this repo)
    │
    ▼
ArgoCD (argocd.homelab.local) — continuous sync, ~30s loop
    │
    ├── Detects drift or new commit
    ├── Runs helm dependency build + helm upgrade --install
    └── Applies to target namespace in k3s cluster
            │
            ├── NFS StorageClass (persistent volumes)
            ├── Traefik Ingress (routing by hostname)
            └── Application pod(s)
```

**Cluster:** k3s on `k8s-ctrl-01, k8s-worker-01/02` (control plane + 2 workers)  
**Ingress:** Traefik at `192.168.1.200` — all apps routed by hostname via single LoadBalancer IP  
**Storage:** NFS StorageClass (default) — provisioned by `nfs-subdir-external-provisioner` against `nfs-01` (`/exports/k8s`)  
**GitOps controller:** ArgoCD at `http://argocd.homelab.local` — credentials managed in Vault  

---

## Repository Structure

```
homelabinfra-k8s-apps/
├── argocd-apps/                    # ArgoCD Application manifests — one per app
│   └── sonarqube.yaml
└── sonarqube/                      # Helm chart wrapper for SonarQube
    ├── Chart.yaml                  # Declares upstream chart as dependency
    └── values.yaml                 # NNT-specific overrides
```

**Pattern — every application follows the same structure:**

```
<appname>/
├── Chart.yaml      # apiVersion: v2, declares upstream chart in dependencies
└── values.yaml     # Overrides for ingress, storage, resources, etc.

argocd-apps/
└── <appname>.yaml  # ArgoCD Application CR — points to <appname>/ in this repo
```

---

## Prerequisites

These must exist in the cluster before any app can deploy successfully:

| Dependency | How It's Deployed |
|---|---|
| k3s cluster (3 nodes) | `setup_k3s.yml` + `setup_k8_worker.yml` Ansible playbooks |
| MetalLB | `setup_metallb.yml` — IP pool 192.168.1.200–210 |
| Traefik ingress | `setup_traefik.yml` — ExternalIP 192.168.1.200 |
| ArgoCD | `setup_argocd.yml` — UI at argocd.homelab.local |
| NFS StorageClass | `k8s_nfs_provisioner` Ansible role — default StorageClass |
| ArgoCD → GitHub credential | `setup_argocd_repo.yml` — K8s secret from Vault PAT |

All Ansible playbooks live in `homelabinfra-iac-ansible`.

---

## Deploying an Application

Applications are self-describing. Applying the ArgoCD Application manifest is the only manual step — after that, ArgoCD takes over.

```bash
# Apply a new application
kubectl apply -f argocd-apps/<appname>.yaml

# Watch sync status
kubectl -n argocd get application <appname>

# Force a sync if needed
kubectl -n argocd patch application <appname> \
  --type merge -p '{"operation":{"sync":{}}}'
```

ArgoCD will then:
1. Clone this repo
2. Run `helm dependency build` (downloads upstream chart)
3. Run `helm upgrade --install` with overrides from `values.yaml`
4. Create the namespace if it doesn't exist (`CreateNamespace=true`)
5. Continue reconciling every ~30 seconds (`selfHeal: true`)

---

## Adding a New Application

**Step 1 — Create the Helm wrapper:**

```yaml
# <appname>/Chart.yaml
apiVersion: v2
name: <appname>
description: <description> — NNT homelab
type: application
version: 1.0.0

dependencies:
  - name: <chart-name>
    version: "<chart-version>"
    repository: <helm-repo-url>
```

```yaml
# <appname>/values.yaml
<chart-name>:
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - name: <appname>.homelab.local
        path: /
  persistence:
    enabled: true
    storageClass: "nfs"
    size: <size>
```

**Step 2 — Create the ArgoCD Application manifest:**

```yaml
# argocd-apps/<appname>.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <appname>
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://github.com/ptchuitio84/homelabinfra-k8s-apps.git
    targetRevision: main
    path: <appname>
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: <appname>
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Step 3 — Add a DNS record:**

`<appname>.homelab.local → 192.168.1.200` (Traefik IP — same for all apps, routed by Host header).

**Step 4 — Apply:**

```bash
git add <appname>/ argocd-apps/<appname>.yaml
git commit -m "feat(<appname>): initial Helm chart + ArgoCD application"
git push
kubectl apply -f argocd-apps/<appname>.yaml
```

---

## Sync Policy Explained

| Setting | Value | Why |
|---|---|---|
| `automated.selfHeal` | `true` | Drift correction — if someone manually changes a resource, ArgoCD restores it |
| `automated.prune` | `false` | Resources removed from Git are **not** auto-deleted. Manual review required before deletion. |
| `CreateNamespace` | `true` | Namespace created automatically on first sync |
| `ServerSideApply` | `true` | Uses Kubernetes server-side apply — handles large objects and field management correctly |

`prune: false` is intentional. Auto-pruning can delete stateful resources (PVCs, secrets) if a config file is moved or renamed. Review and delete manually.

---

## Current Applications

| App | Namespace | Hostname | Chart | Storage |
|---|---|---|---|---|
| SonarQube | sonarqube | sonarqube.homelab.local | sonarqube 10.7.0 | NFS, 20Gi data + 20Gi DB |

---

## SonarQube

**Access:** `http://sonarqube.homelab.local`  
**Default credentials:** admin / admin (change on first login)  
**Edition:** Community (free, single-branch analysis)  
**Database:** Embedded PostgreSQL (acceptable for lab — replace with external DB before any production use)  
**JVM:** `-Xms512m -Xmx1024m` (1GB heap, 2GB container limit)

SonarQube is used for static code analysis on the AIgentic Solutions project and any other repo linked to the Jenkins pipeline.

---

## Design Decisions

**Why umbrella Helm charts instead of vendoring?**
The upstream chart (`sonarqube:10.7.0`) is declared as a dependency in `Chart.yaml`. ArgoCD runs `helm dependency build` at sync time — no chart files are committed here. Upgrading means bumping the version in `Chart.yaml` and pushing. Nothing to vendor, nothing to diff against.

**Why ArgoCD over pure Helm pipeline?**
ArgoCD provides continuous reconciliation, not just one-shot deploy. Manual changes (emergency hotfixes, operator edits) are caught and corrected. Every deployment is reflected in the UI with history, rollback, and diff visibility. Pipeline deploys are fire-and-forget; ArgoCD is a control loop.

**Why `prune: false`?**
Stateful workloads (PVCs, DB data) should never be auto-deleted based on a Git change. Deletion requires deliberate intent — remove the manifest and explicitly run `argocd app delete` or `kubectl delete`.
