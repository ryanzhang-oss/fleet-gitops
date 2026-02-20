# Fleet GitOps

This repository demonstrates how to use [KubeFleet](https://github.com/Azure/fleet) with GitOps tools to deploy and progressively roll out applications across multiple Kubernetes clusters. It contains two end-to-end examples, each using a different GitOps engine.

## Before You Begin

These demos assume working knowledge of the following technologies:

- **[KubeFleet](https://github.com/Azure/fleet)** — Multi-cluster orchestration for Kubernetes. Distributes workloads across a fleet of clusters using placements, overrides, and staged rollout strategies.
- **[Kustomize](https://kustomize.io/)** — Kubernetes-native configuration management for customizing YAML manifests without templating.
- **[Flux CD](https://fluxcd.io/)** — A GitOps toolkit that continuously reconciles cluster state with declarations in Git.
- **[Argo CD](https://argo-cd.readthedocs.io/)** — A declarative GitOps continuous delivery tool for Kubernetes.

If you are new to any of these, review their official documentation before proceeding.

### Fork the Repositories

To practice GitOps workflows hands-on, fork the following repositories into your own GitHub account:

| Repository | Description |
|------------|-------------|
| **[fleet-gitops](https://github.com/ryanzhang-oss/fleet-gitops)** (this repo) | Fleet placements, overrides, staged rollout strategies, and GitOps controller configurations. |
| **[guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo)** | Application manifests and per-cluster overlays/configurations consumed by the GitOps controllers. |

After forking, update the Git URLs in the YAML files throughout this repository to point to your forks. This allows the GitOps controllers (Flux or Argo CD) to watch your repositories so you can experiment with making changes, opening pull requests, and observing reconciliation in action.

## Applications

### 1. NGINX via Flux CD and KubeFleet

Uses **Flux CD** on member clusters with **Kustomize overlays** for per-cluster configuration.

| | |
|---|---|
| **GitOps engine** | Flux CD |
| **Application source** | [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) repo, `kustomize-nginx/` directory |
| **Hub configuration** | `rollout/kustomize-nginx/` |
| **Application resources** | `applications/kustomize-nginx/` |
| **Rollout strategy** | Staged: staging → canary → prod with approval gates |
| **Full guide** | [rollout/kustomize-nginx/README.md](rollout/kustomize-nginx/README.md) |

### 2. Guestbook via Argo CD and KubeFleet

Uses **Argo CD** on member clusters with per-cluster application paths.

| | |
|---|---|
| **GitOps engine** | Argo CD |
| **Application source** | [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) repo, `guestbook/` directory |
| **Hub configuration** | `rollout/guestbook/` |
| **Application resources** | `applications/simple-guestbook/` |
| **Rollout strategy** | Staged: staging → canary → prod with approval gates |
| **Full guide** | [rollout/guestbook/README.md](rollout/guestbook/README.md) |

## Repository Structure

```
fleet-gitops/
├── applications/
│   ├── kustomize-nginx/        # Flux CRs, Fleet placements, overrides, and rollout
│   │   │                         resources synced to the hub by Flux
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── serviceaccount.yaml
│   │   ├── rbac.yaml
│   │   ├── git-repository.yaml
│   │   ├── kustomization-nginx-application.yaml
│   │   ├── application-placement.yaml
│   │   ├── nginx-ns-placement.yaml
│   │   ├── application-override.yaml
│   │   ├── updateStrategy.yaml
│   │   ├── updateRun-1.yaml
│   │   └── updateRun-2.yaml
│   └── simple-guestbook/       # Argo CD CRs, Fleet placements, overrides, and rollout
│       │                         resources synced to the hub by Argo CD
│       ├── simple-guestbook-application.yaml
│       ├── guestbook-appProj.yaml
│       ├── application-placement.yaml
│       ├── appProject-placement.yaml
│       ├── guestbook-ns-placement.yaml
│       ├── application-override.yaml
│       ├── updateStrategy.yaml
│       └── updateRun.yaml
├── rollout/
│   ├── kustomize-nginx/        # Hub-level Flux bootstrap for the NGINX demo
│   │   ├── README.md
│   │   ├── git-repository.yaml
│   │   └── kustomization-nginx.yaml
│   └── guestbook/              # Hub-level Argo CD bootstrap for the Guestbook demo
│       ├── README.md
│       ├── guestbook-appProj.yaml
│       └── guestbook-hub-manifests.yaml
└── LICENSE
```

## How It Works

Both demos follow the same pattern:

1. **Bootstrap the hub cluster.** Apply the resources in the corresponding `rollout/` directory to configure the GitOps controller (Flux or Argo CD) on the hub cluster. This controller watches a directory in this repository and syncs its contents to the hub.
2. **Sync application resources to the hub.** The GitOps controller reconciles all resources from the corresponding `applications/` directory onto the hub cluster — including Fleet placements, overrides, the application CRs, and the staged rollout definitions.
3. **Distribute to member clusters.** KubeFleet `ResourcePlacement` and `ClusterResourcePlacement` resources propagate the GitOps CRs (Flux Kustomization or Argo CD Application), RBAC, and namespaces to member clusters.
4. **Per-cluster customization.** `ResourceOverride` applies JSON patches to the GitOps CRs on each member cluster, pointing them to cluster-specific paths/overlays and activating reconciliation.
5. **Staged rollout.** A `StagedUpdateStrategy` defines the progression (staging → canary → prod) with concurrency limits and approval gates. A `StagedUpdateRun` executes the rollout.
6. **GitOps on member clusters.** The GitOps controller on each member cluster pulls and applies the application manifests from the specified repository and path, deploying the application with cluster-specific configurations.

## Getting Started

Pick either demo and follow its detailed guide:

- **Flux + Kustomize:** [rollout/kustomize-nginx/README.md](rollout/kustomize-nginx/README.md)
- **Argo CD:** [rollout/guestbook/README.md](rollout/guestbook/README.md)
