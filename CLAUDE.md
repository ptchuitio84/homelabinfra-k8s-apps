# homelabinfra-k8s-apps

ArgoCD GitOps for k8s workloads on the NNT k3s cluster.
ArgoCD watches main branch and syncs on every push.

## Pattern — Umbrella Helm Chart
Each app follows this structure:
```
<appname>/
  Chart.yaml        # declares upstream chart as dependency
  values.yaml       # homelab overrides only
  templates/        # additional k8s resources not in upstream chart
argocd-apps/
  <appname>.yaml    # ArgoCD Application CR
```
Nothing vendored into git. ArgoCD runs `helm dependency build` then `helm upgrade`.

## Cluster
- k3s v1.32 on OL9.7
- Control plane: hmvlapk8s001 (10.10.1.61)
- Workers: hmvlapk8s002 (10.10.1.45), hmvlapk8s003 (10.10.1.46)
- Default storageClass: `nfs` (local-path default was removed)
- Ingress: Traefik at 10.10.1.200

## Key Services
| App | Namespace | URL |
|-----|-----------|-----|
| ArgoCD | argocd | argocd.nnt.com |
| SonarQube | sonarqube | sonarqube.nnt.com |
| Traefik | traefik | 10.10.1.200 |

## Known Gotchas
- **ArgoCD Redis cache:** When ArgoCD loops on a stale error after a fix is pushed, restart alone is not enough. Must flush Redis `mfst|...|<commit>|...` keys manually. `argocd.argoproj.io/refresh: hard` does NOT flush manifest cache.
- **SonarSource 2026.x chart:** `edition: "community"` is invalid. Use `community.enabled: true`. `monitoringPasscode` is required.
- **postgres on NFS:** Requires busybox initContainer to chown UID 999. Set `PGDATA` to a subdirectory of the mount point.
- **Bitnami images:** Do not use bitnami PostgreSQL. Images don't exist on Docker Hub for PG17+. Use `postgres:17-alpine` (Docker Official Images).
- **MetalLB/webhook pods:** Must run on control plane (`nodeSelector: node-role.kubernetes.io/control-plane: "true"`). k3s API server can't reach webhooks on workers.
- **firewalld:** Disabled on all k3s nodes. k3s owns iptables — they cannot coexist.
