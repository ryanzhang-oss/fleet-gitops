# Manage NGINX Application via Flux and KubeFleet

This demo shows how to use Fleet to roll out the NGINX application (defined in the [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) repository) across multiple Kubernetes clusters with staged rollouts.

## Before You Begin

This guide assumes familiarity with the following technologies:

- **[Kustomize](https://kustomize.io/)** — A Kubernetes-native configuration management tool used to customize YAML manifests without templating. This demo uses Kustomize overlays to apply per-cluster configurations.
- **[Flux CD](https://fluxcd.io/)** — A GitOps toolkit for Kubernetes that continuously reconciles cluster state with declarations in Git. Flux watches Git repositories and automatically applies changes to clusters.
- **[KubeFleet](https://github.com/Azure/fleet)** — A multi-cluster orchestration system that distributes and manages workloads across a fleet of Kubernetes clusters using placements, overrides, and staged rollout strategies.

If you are new to any of these, review their official documentation before proceeding.

### Fork the Repositories

To follow along and practice GitOps workflows hands-on, fork both repositories into your own GitHub account:

1. **This repository:** [fleet-gitops](https://github.com/ryanzhang-oss/fleet-gitops) — contains the Fleet placement, override, and rollout resources as well as the Flux configuration that ties everything together.
2. **The application repository:** [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) — contains the NGINX application manifests and per-cluster Kustomize overlays.

After forking, update the Git URLs in the YAML files (e.g., `git-repository.yaml`) to point to your forks so that Flux watches your repositories and you can experiment with making changes, opening pull requests, and observing GitOps reconciliation in action.

## Prerequisites

- A Kubernetes hub cluster with Fleet installed
- One or more member clusters joined to the hub cluster via Fleet
- Member clusters labeled with `kubernetes-fleet.io/env` (values: `staging`, `canary`, or `prod`)
- Flux CD controller installed on all member clusters
- Azure CLI (if using the `az` command method)

## Install and Configure Flux on the Hub Cluster

The following command installs Flux CD on the hub cluster and configures it to sync resources from the `applications/kustomize-nginx/` directory in the `fleet-gitops` repository:

```bash
az k8s-configuration flux create --resource-group ryanzhang-testgroup --cluster-name ryan-hub --cluster-type managedClusters --name nginx-app --scope namespace --namespace  flux-kustomize   --kind git --url https://github.com/ryanzhang-oss/fleet-gitops/   --branch main --kustomization name=nginx-kustomization path=applications/kustomize-nginx prune=true
```

**Alternative:** Instead of using the Azure CLI, you can manually install Flux CD and apply the configuration by running:

```bash
kubectl apply -f rollout/kustomize-nginx/git-repository.yaml
kubectl apply -f rollout/kustomize-nginx/kustomization-nginx.yaml
```

Both methods install Flux CD on the hub cluster and configure it to sync resources from the `applications/kustomize-nginx/` directory.

## Roll Out the NGINX Application to Member Clusters via KubeFleet

The custom resources in the `applications/kustomize-nginx/` directory instruct the Flux controller to apply all the YAML files in that directory to the hub cluster.

Below is a summary of each file and its role in deploying and rolling out the NGINX application across fleet member clusters.

### Orchestration

| File | Description |
|------|-------------|
| **kustomization.yaml** | The Kustomize entry point that lists all of the below resources and sets the default namespace to `nginx-demo`. This is the file that the Flux configuration on the hub cluster points to. |

### Infrastructure, RBAC & Flux GitOps

| File | Description |
|------|-------------|
| **namespace.yaml** | Creates the `nginx-demo` namespace where all NGINX-related resources reside. |
| **serviceaccount.yaml** | Defines the `flux-applier` ServiceAccount used by the Flux Kustomize controller to apply resources on member clusters. |
| **rbac.yaml** | Contains a `Role` and `RoleBinding` that grant the `flux-applier` ServiceAccount permissions to apply the application to the target namespace. |
| **git-repository.yaml** | A Flux `GitRepository` source pointing to the `guestbook-demo` repository (tag `v0.4.2`). This is the upstream Git source that Flux watches for changes at a 1-minute interval. |
| **kustomization-nginx-application.yaml** | A Flux `Kustomization` resource that tells Flux to reconcile manifests from the path `./kustomize-nginx/overlays/*` in the `GitRepository` source into the `nginx-demo` namespace. It is created in a **suspended** state (`suspend: true`) so that it does not reconcile until the per-cluster override activates it. |

### KubeFleet Placement & Override

| File | Description |
|------|-------------|
| **application-placement.yaml** | A `ResourcePlacement` (namespaced) with a `PickAll` policy that selects the Flux `Kustomization`, `GitRepository`, RBAC `Role`/`RoleBinding`, and `ServiceAccount` resources and distributes them to all matching member clusters. It uses an **External** rollout strategy, meaning rollout is controlled by a `StagedUpdateRun` rather than automatic rolling updates. These are the actual Flux CRs that will sync the application to the member clusters. |
| **nginx-ns-placement.yaml** | A `ClusterResourcePlacement` that propagates the `nginx-demo` `Namespace` (namespace-only scope) to all member clusters. |
| **application-override.yaml** | A `ResourceOverride` tied to the `kustomized-ngix` placement. For each member cluster, it applies two JSON-patch operations on the Flux `Kustomization`: (1) replaces the `spec.path` with a cluster-specific overlay path (`./kustomize-nginx/overlays/${MEMBER-CLUSTER-NAME}`), enabling per-cluster configuration, and (2) sets `spec.suspend` to `false` to activate reconciliation. |

### Staged Rollout

| File | Description |
|------|-------------|
| **updateStrategy.yaml** | A `StagedUpdateStrategy` named `nginx-update-strategy` that defines a three-stage rollout: **staging** → **canary** → **prod**. Each stage selects clusters by label (`kubernetes-fleet.io/env`). Staging allows 1 concurrent update with a 1-minute timed wait and manual approval. Canary allows 50% concurrency. Prod allows 30% concurrency with a pre-stage approval gate. |
| **updateRun-1.yaml** | A `StagedUpdateRun` named `nginx-release-1` that references the `kustomized-ngix` placement and the `nginx-update-strategy`. Setting `state: Run` kicks off the staged rollout process. |

## How It All Fits Together

After you run the command (or apply the YAML files in `rollout/kustomize-nginx/` to configure Flux), here is how the deployment and rollout process unfolds:

1. **Flux on the hub cluster** reconciles `kustomization.yaml` in the `applications/kustomize-nginx` directory, which creates all the resources in the `nginx-demo` namespace on the hub.
2. **KubeFleet placements** (`application-placement.yaml` and `nginx-ns-placement.yaml`) propagate the namespace, Flux GitRepository, Kustomization, RBAC, and ServiceAccount to member clusters.
3. The **ResourceOverride** customizes the Flux Kustomization per member cluster, pointing each to its own overlay path and un-suspending it so Flux begins deploying NGINX.
4. The **StagedUpdateStrategy** and **StagedUpdateRun** orchestrate the rollout across staging, canary, and production clusters with approval gates, ensuring safe progressive delivery.
5. **Flux on the member clusters** pulls the manifests from the specified Git repository and path directed by the Flux GitRepository/Kustomization placed and rolled out by KubeFleet. In this way, the NGINX application is deployed with any cluster-specific customizations defined in the overlays in the `guestbook-demo` repository.

## Continuous Deployment

To roll out an updated version of the NGINX application, complete the following steps:

1. **Update the application manifests.** Make the desired changes in the `guestbook-demo` repository (for example, update the container image tag in the Deployment). Per-cluster customizations can also be applied by editing the corresponding Kustomize overlay. Open a pull request, review, and merge to the main branch.
2. **Create a new release.** Tag a new release or create a release branch in the `guestbook-demo` repository to mark the updated version.
3. **Point to the new release.** In this repository, update `applications/kustomize-nginx/git-repository.yaml` to reference the new tag or branch.
4. **Add a new StagedUpdateRun.** Create a new `StagedUpdateRun` manifest (for example, `updateRun-3.yaml`) that references the same placement and update strategy with a unique name (e.g., `nginx-release-2`). Set `state: Initialize` to ensure you can examine the rollout details before it begins.
5. **Push the changes.** Update `applications/kustomize-nginx/kustomization.yaml` to reference the new `StagedUpdateRun` manifest. Commit and push to the `main` branch of this repository.
6. **Automatic reconciliation.** The Flux configuration on the hub cluster detects the updated `GitRepository` reference and triggers a new reconciliation cycle.
7. **Review the rollout plan.** The new `StagedUpdateRun` in `Initialize` state generates a detailed rollout plan showing which clusters are assigned to each stage. Review the plan and resource snapshots to confirm you are rolling out what is intended and that the stage assignments and progression order are correct before proceeding.
8. **Approve and monitor.** Set the `StagedUpdateRun` to `state: Run` to start the rollout (either in the repo or via kubectl). Monitor progress and approve each stage as needed until the new version is fully deployed to all production clusters.
