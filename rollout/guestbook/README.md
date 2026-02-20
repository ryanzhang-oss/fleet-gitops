# Manage Guestbook Application via Argo CD and KubeFleet

This demo shows how to use KubeFleet to roll out the Guestbook application (defined in the [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) repository) across multiple Kubernetes clusters with staged rollouts, using Argo CD as the GitOps engine.

## Before You Begin

This guide assumes familiarity with the following technologies:

- **[Argo CD](https://argo-cd.readthedocs.io/)** — A declarative GitOps continuous delivery tool for Kubernetes. Argo CD watches Git repositories and automatically syncs application state to clusters using Application and AppProject custom resources.
- **[KubeFleet](https://github.com/Azure/fleet)** — A multi-cluster orchestration system that distributes and manages workloads across a fleet of Kubernetes clusters using placements, overrides, and staged rollout strategies.

If you are new to either of these, review their official documentation before proceeding.

### Fork the Repositories

To follow along and practice GitOps workflows hands-on, fork both repositories into your own GitHub account:

1. **This repository:** [fleet-gitops](https://github.com/ryanzhang-oss/fleet-gitops) — contains the Fleet placement, override, and rollout resources as well as the Argo CD configuration that ties everything together.
2. **The application repository:** [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) — contains the Guestbook application manifests and per-cluster configurations.

After forking, update the Git URLs in the YAML files (e.g., `simple-guestbook-application.yaml`, `guestbook-hub-manifests.yaml`) to point to your forks so that Argo CD watches your repositories and you can experiment with making changes, opening pull requests, and observing GitOps reconciliation in action.

## Prerequisites

- A Kubernetes hub cluster with Fleet installed
- One or more member clusters joined to the hub cluster via Fleet
- Member clusters labeled with `kubernetes-fleet.io/env` (values: `staging`, `canary`, or `prod`)
- Argo CD installed on the hub cluster and all member clusters

## Install and Configure Argo CD on the Hub Cluster

Apply the hub-level Argo CD resources to configure the hub cluster to sync the Guestbook application resources from this repository:

```bash
kubectl apply -f rollout/guestbook/guestbook-appProj.yaml
kubectl apply -f rollout/guestbook/guestbook-hub-manifests.yaml
```

This creates:

1. An Argo CD **AppProject** (`project-gitops`) that authorizes syncing Fleet placement CRDs and application resources into the `guestbook` and `argocd` namespaces.
2. An Argo CD **Application** (`guestbook-hub-manifests`) that watches the `applications/simple-guestbook` directory in this repository and automatically syncs all resources to the hub cluster.

## Roll Out the Guestbook Application to Member Clusters via KubeFleet

The Argo CD Application on the hub cluster reconciles all the YAML files in the `applications/simple-guestbook/` directory. Below is a summary of each file and its role in deploying and rolling out the Guestbook application across fleet member clusters.

### Argo CD GitOps

| File | Description |
|------|-------------|
| **guestbook-appProj.yaml** | An Argo CD `AppProject` named `project-guestbook` in the `argocd` namespace. It restricts the Application to deploying only into the `guestbook` namespace and only from the `guestbook-demo` Git repository. This AppProject is distributed to member clusters so Argo CD on each cluster can manage the application. |
| **simple-guestbook-application.yaml** | An Argo CD `Application` named `guestbook-simple-app` in the `guestbook` namespace. It points to the `guestbook-demo` repository (tag `v0.3.0`) at path `guestbook/*`. Auto-sync is initially **disabled** (`enabled: false`) so that reconciliation does not begin until the per-cluster override activates it. Includes retry and prune policies for robust syncing. |

### KubeFleet Placement & Override

| File | Description |
|------|-------------|
| **application-placement.yaml** | A `ResourcePlacement` with a `PickAll` policy that selects the Argo CD `Application` resource and distributes it to all matching member clusters. It uses an **External** rollout strategy, meaning rollout is controlled by a `StagedUpdateRun` rather than automatic rolling updates. |
| **appProject-placement.yaml** | A `ResourcePlacement` with a `PickAll` policy that distributes the Argo CD `AppProject` to all member clusters using a **RollingUpdate** strategy (max 1 unavailable, 25% surge). |
| **guestbook-ns-placement.yaml** | A `ClusterResourcePlacement` that propagates the `guestbook` `Namespace` to all member clusters using a RollingUpdate strategy. |
| **application-override.yaml** | A `ResourceOverride` tied to the `guestbook-app` placement. For each member cluster, it applies two JSON-patch operations on the Argo CD `Application`: (1) replaces `spec.source.path` with a cluster-specific path (`guestbook/${MEMBER-CLUSTER-NAME}`), enabling per-cluster configuration, and (2) sets `spec.syncPolicy.automated.enabled` to `true` to activate auto-sync. |

### Staged Rollout

| File | Description |
|------|-------------|
| **updateStrategy.yaml** | A `StagedUpdateStrategy` named `guestbook-strategy` that defines a three-stage rollout: **staging** → **canary** → **prod**. Each stage selects clusters by label (`kubernetes-fleet.io/env`). Staging allows 1 concurrent update with a 1-minute timed wait and manual approval. Canary allows 50% concurrency with post-stage approval. Prod allows 30% concurrency with a pre-stage approval gate. |
| **updateRun.yaml** | A `StagedUpdateRun` named `guestbook-release-v0.3.0` that references the `guestbook-app` placement and the `guestbook-strategy`. Setting `state: Run` kicks off the staged rollout process. |

## How It All Fits Together

After you apply the YAML files in `rollout/guestbook/` to configure Argo CD on the hub cluster, here is how the deployment and rollout process unfolds:

1. **Argo CD on the hub cluster** syncs the `applications/simple-guestbook` directory, creating all the resources (Application, AppProject, placements, overrides, rollout strategy) in the `guestbook` and `argocd` namespaces on the hub.
2. **KubeFleet placements** (`guestbook-ns-placement.yaml`, `appProject-placement.yaml`, and `application-placement.yaml`) propagate the namespace, AppProject, and Application to member clusters.
3. The **ResourceOverride** customizes the Argo CD Application per member cluster, pointing each to its own application path (`guestbook/${MEMBER-CLUSTER-NAME}`) and enabling auto-sync so Argo CD begins deploying the Guestbook app.
4. The **StagedUpdateStrategy** and **StagedUpdateRun** orchestrate the rollout across staging, canary, and production clusters with approval gates, ensuring safe progressive delivery.
5. **Argo CD on the member clusters** syncs the manifests from the specified Git repository and path directed by the Application placed and rolled out by KubeFleet. The Guestbook application is deployed with any cluster-specific customizations.

## Continuous Deployment

To roll out an updated version of the Guestbook application, complete the following steps:

1. **Update the application manifests.** Make the desired changes in the `guestbook-demo` repository (for example, update the container image tag in the Deployment). Per-cluster customizations can also be applied by editing the corresponding directory. Open a pull request, review, and merge to the main branch.
2. **Create a new release.** Tag a new release in the `guestbook-demo` repository to mark the updated version.
3. **Point to the new release.** In this repository, update `applications/simple-guestbook/simple-guestbook-application.yaml` to reference the new tag in `spec.source.targetRevision`.
4. **Add a new StagedUpdateRun.** Create a new `StagedUpdateRun` manifest that references the same placement and update strategy with a unique name (e.g., `guestbook-release-v0.4.0`). Set `state: Initialize` to examine the rollout details before it begins.
5. **Push the changes.** Commit and push to the `main` branch of this repository.
6. **Automatic reconciliation.** The Argo CD Application on the hub cluster detects the changes and triggers a new sync cycle.
7. **Review the rollout plan.** The new `StagedUpdateRun` in `Initialize` state generates a detailed rollout plan showing which clusters are assigned to each stage. Review the plan to confirm stage assignments and progression order.
8. **Approve and monitor.** Set the `StagedUpdateRun` to `state: Run` to start the rollout (either in the repo or via kubectl). Monitor progress and approve each stage as needed until the new version is fully deployed to all production clusters.
