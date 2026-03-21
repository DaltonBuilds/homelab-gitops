# homelab-gitops

Kubernetes manifests, Helm values, and GitOps configuration for my homelab — covering the workload cluster (k3s + ArgoCD) and management cluster (k3s + Flux).

This is one of three repos that make up my homelab infrastructure. [homelab-terraform](https://github.com/DaltonBuilds/homelab-terraform) provisions the VMs, [homelab-ansible](https://github.com/DaltonBuilds/homelab-ansible) configures the OS, and this repo drives everything running in Kubernetes.

## Clusters

**Workload cluster** — managed by ArgoCD using an app-of-apps pattern. Four nodes: gandalf (control plane) + aragorn, legolas, gimli (workers).

**Management cluster** — single-node k3s on a dedicated VM (`mgmt-plane`), managed by Flux. Runs the observability stack independently from the workload cluster so it stays up during rebuilds.

## Stack

| Layer | Tool |
|---|---|
| CNI | Cilium (eBPF, replaces kube-proxy + Flannel) |
| Load balancing | Cilium L2 announcements (replaces MetalLB) |
| Ingress | Cilium Gateway API |
| GitOps (workload) | ArgoCD with app-of-apps |
| GitOps (mgmt) | Flux |
| TLS | cert-manager with DNS-01 via Cloudflare |
| Secrets | External Secrets Operator + GCP Secret Manager |
| Storage (file) | NFS provisioner backed by ZFS on dedicated VM |
| Storage (object) | Garage (S3-compatible) |
| Observability | Prometheus, Loki, Grafana, Alloy, Alertmanager |
| Tunnel | Cloudflared |

## Structure

```
homelab-gitops/
├── argocd/
│   ├── root.yaml              # App-of-apps root application
│   └── apps/                  # ArgoCD Application manifests
└── apps/
    ├── cilium/                # L2 IP pool, L2AnnouncePolicy
    ├── cert-manager/          # ClusterIssuer, Cloudflare secret
    ├── external-secrets/      # ClusterSecretStore (GCP)
    ├── nfs-provisioner/       # StorageClass definitions
    └── platform-ingress/      # HTTPRoute/Ingress for internal tools
```

## App-of-Apps

ArgoCD is bootstrapped by applying `argocd/root.yaml` after initial install. The root app watches `argocd/apps/` and creates child Applications for each component. All apps sync from this repo, so the cluster state is fully recoverable from Git.

## Secrets

Secrets are not stored in this repo. External Secrets Operator pulls secrets from GCP Secret Manager at runtime using a `ClusterSecretStore`. The only secret committed is a placeholder reference — actual values live in GCP.
