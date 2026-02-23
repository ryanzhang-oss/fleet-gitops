# Fleet GitOps

GitOps configuration repository for multi-cluster Kubernetes application deployment using [KubeFleet](https://github.com/Azure/fleet). This repository contains declarative resources for distributing and managing workloads across a fleet of Kubernetes clusters with staged rollout strategies.

## Overview

This repository demonstrates how to combine KubeFleet with GitOps tools (Flux CD and Argo CD) to achieve progressive, multi-cluster application delivery. Each example shows how to:

- Distribute application resources from a hub cluster to member clusters via KubeFleet placements
- Customize deployments per cluster using overrides
- Perform staged rollouts across staging, canary, and production environments with approval gates

## Repository Structure

```
fleet-gitops/
├── applications/                    # Application resources synced to the hub cluster
│   ├── kustomize-nginx/             # NGINX app using Flux CD + Kustomize
│   └── simple-guestbook/            # Guestbook app using Argo CD
├── rollout/                         # Hub-level bootstrap manifests
│   ├── kustomize-nginx/             # Flux GitRepository & Kustomization for NGINX
│   └── guestbook/                   # Argo CD AppProject & Application for Guestbook
└── LICENSE
```

### `applications/`

Contains the KubeFleet placement, override, staged rollout, and GitOps resources that Flux or Argo CD syncs onto the hub cluster. These resources instruct KubeFleet to distribute and manage the applications across member clusters.

### `rollout/`

Contains the bootstrap manifests applied directly to the hub cluster to set up the GitOps pipeline (Flux or Argo CD) that watches and syncs the `applications/` directory.

## Examples

### NGINX via Flux CD + Kustomize

Deploys an NGINX application across multiple clusters using Flux CD for GitOps reconciliation and Kustomize overlays for per-cluster configuration. Features staged rollouts through staging, canary, and production environments.

- **Hub bootstrap:** [`rollout/kustomize-nginx/`](rollout/kustomize-nginx/)
- **Application resources:** [`applications/kustomize-nginx/`](applications/kustomize-nginx/)
- **Detailed guide:** [`rollout/kustomize-nginx/README.md`](rollout/kustomize-nginx/README.md)

### Guestbook via Argo CD

Deploys a Guestbook application across multiple clusters using Argo CD for GitOps reconciliation. Features per-cluster path overrides and staged rollouts through staging, canary, and production environments.

- **Hub bootstrap:** [`rollout/guestbook/`](rollout/guestbook/)
- **Application resources:** [`applications/simple-guestbook/`](applications/simple-guestbook/)
- **Detailed guide:** [`rollout/guestbook/README.md`](rollout/guestbook/README.md)

## Prerequisites

- A Kubernetes hub cluster with [KubeFleet](https://github.com/Azure/fleet) installed
- One or more member clusters joined to the hub cluster via Fleet
- Member clusters labeled with `kubernetes-fleet.io/env` (values: `staging`, `canary`, or `prod`)
- **For the NGINX example:** [Flux CD](https://fluxcd.io/) installed on the hub and all member clusters
- **For the Guestbook example:** [Argo CD](https://argo-cd.readthedocs.io/) installed on the hub and all member clusters

## Getting Started

1. Fork this repository and the companion [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) repository.
2. Update Git URLs in the YAML files to point to your forks.
3. Choose an example and follow its detailed guide:
   - [NGINX via Flux CD](rollout/kustomize-nginx/README.md)
   - [Guestbook via Argo CD](rollout/guestbook/README.md)

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
