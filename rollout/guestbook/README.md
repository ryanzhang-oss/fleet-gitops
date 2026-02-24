# Manage Guestbook Application via Argo CD and KubeFleet

This demo shows how to use Fleet to roll out a simple Guestbook application (defined in the [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) repository) across multiple Kubernetes clusters with staged rollouts.

## Before You Begin

This guide assumes familiarity with the following technologies:

- **[Argo CD](https://argo-cd.readthedocs.io/)** — A declarative, GitOps continuous delivery tool for Kubernetes. Argo CD watches Git repositories and automatically syncs application state to clusters.
- **[KubeFleet](https://github.com/Azure/fleet)** — A multi-cluster orchestration system that distributes and manages workloads across a fleet of Kubernetes clusters using placements, overrides, and staged rollout strategies.

If you are new to any of these, review their official documentation before proceeding.

### Fork the Repositories

To follow along and practice GitOps workflows hands-on, fork both repositories into your own GitHub account:

1. **This repository:** [fleet-gitops](https://github.com/ryanzhang-oss/fleet-gitops) — contains the Fleet placement, override, and rollout resources as well as the Argo CD configuration that ties everything together.
2. **The application repository:** [guestbook-demo](https://github.com/ryanzhang-oss/guestbook-demo) — contains the Guestbook application manifests and per-stage configurations (organized by environment label, e.g., `staging`, `canary`, `prod`).

After forking, update the Git URLs in the YAML files (e.g., `simple-guestbook-application.yaml`, `guestbook-appProj.yaml`) to point to your forks so that Argo CD watches your repositories and you can experiment with making changes, opening pull requests, and observing GitOps reconciliation in action.

## Prerequisites

- A Kubernetes hub cluster with Fleet installed
- One or more member clusters joined to the hub cluster via Fleet
- Member clusters labeled with `kubernetes-fleet.io/env` (values: `staging`, `canary`, or `prod`)
- Argo CD installed on the hub cluster and all member clusters

## Install and Configure Argo CD on the Hub Cluster

Apply the hub-level manifests from this directory to set up the Argo CD `AppProject` and `Application` that sync fleet resources from the `applications/simple-guestbook/` directory:

```bash
kubectl apply -f rollout/guestbook/guestbook-appProj.yaml
kubectl apply -f rollout/guestbook/guestbook-hub-manifests.yaml
```

The `guestbook-appProj.yaml` creates an Argo CD `AppProject` named `project-gitops` in the `argocd` namespace that whitelists KubeFleet placement resources and permits deployments to the `guestbook` namespace.

The `guestbook-hub-manifests.yaml` creates the `guestbook` namespace and an Argo CD `Application` that syncs resources from the `applications/simple-guestbook/` directory in this repository. The Application is configured with automated sync, pruning, self-healing, and retry policies.

## Roll Out the Guestbook Application to Member Clusters via KubeFleet

Once the hub-level Argo CD Application syncs, it applies all the resources defined in the `applications/simple-guestbook/` directory to the hub cluster.

Below is a summary of each file and its role in deploying and rolling out the Guestbook application across fleet member clusters.

### Argo CD Configuration

| File | Description |
|------|-------------|
| **guestbook-appProj.yaml** | An Argo CD `AppProject` named `project-guestbook` that permits deployments to the `guestbook` namespace and restricts the source to the `guestbook-demo` repository. |
| **simple-guestbook-application.yaml** | An Argo CD `Application` that syncs manifests from the path `guestbook/*` in the `guestbook-demo` repository (tag `v0.3.0`) into the `guestbook` namespace. It is created with automated sync **disabled** (`enabled: false`) so that it does not reconcile until the per-cluster override activates it. |

### KubeFleet Placement & Override

| File | Description |
|------|-------------|
| **appProject-placement.yaml** | A `ResourcePlacement` (namespaced) with a `PickAll` policy that selects the Argo CD `AppProject` resource and distributes it to all member clusters using a `RollingUpdate` strategy. |
| **application-placement.yaml** | A `ResourcePlacement` (namespaced) with a `PickAll` policy that selects the Argo CD `Application` resource and distributes it to all matching member clusters. It uses an **External** rollout strategy, meaning rollout is controlled by a `StagedUpdateRun` rather than automatic rolling updates. |
| **guestbook-ns-placement.yaml** | A `ClusterResourcePlacement` that propagates the `guestbook` `Namespace` (namespace-only scope) to all member clusters using a `RollingUpdate` strategy. |
| **application-override.yaml** | A `ResourceOverride` tied to the `guestbook-app` placement. It applies two JSON-patch operations on the Argo CD `Application`: (1) replaces the `spec.source.path` with a stage-specific path (`guestbook/${MEMBER-CLUSTER-LABEL-KEY-kubernetes-fleet.io/env}`), enabling per-stage configuration based on the cluster's `env` label (e.g., `staging`, `canary`, or `prod`), and (2) sets `spec.syncPolicy.automated.enabled` to `true` to activate reconciliation. |

### Staged Rollout

| File | Description |
|------|-------------|
| **updateStrategy.yaml** | A `StagedUpdateStrategy` named `guestbook-strategy` that defines a three-stage rollout: **staging** -> **canary** -> **prod**. Each stage selects clusters by label (`kubernetes-fleet.io/env`). Staging allows 1 concurrent update with a 1-minute timed wait and manual approval. Canary allows 50% concurrency with manual approval. Prod allows 30% concurrency with a pre-stage approval gate. |
| **updateRun.yaml** | A `StagedUpdateRun` named `guestbook-release-v0.3.0` that references the `guestbook-app` placement and the `guestbook-strategy`. Setting `state: Run` kicks off the staged rollout process. |

## How It All Fits Together

After you apply the hub-level manifests (or configure Argo CD), here is how the deployment and rollout process unfolds:

1. **Argo CD on the hub cluster** syncs the `applications/simple-guestbook` directory, which creates all the resources in the `guestbook` namespace on the hub including the Argo CD Application, AppProject, KubeFleet placements, overrides, and staged rollout resources.
2. **KubeFleet placements** (`appProject-placement.yaml`, `application-placement.yaml`, and `guestbook-ns-placement.yaml`) propagate the namespace, Argo CD AppProject, and Application to member clusters.
3. The **ResourceOverride** customizes the Argo CD Application per stage, pointing each cluster to a stage-specific path (`guestbook/${MEMBER-CLUSTER-LABEL-KEY-kubernetes-fleet.io/env}`, resolved from the cluster's `kubernetes-fleet.io/env` label) and enabling automated sync so Argo CD begins deploying the Guestbook app.
4. The **StagedUpdateStrategy** and **StagedUpdateRun** orchestrate the rollout across staging, canary, and production clusters with approval gates, ensuring safe progressive delivery.
5. **Argo CD on the member clusters** syncs the application manifests from the specified Git repository and stage-specific path directed by the Argo CD Application placed and rolled out by KubeFleet. In this way, the Guestbook application is deployed with any stage-specific customizations defined in the corresponding stage directory of the `guestbook-demo` repository.

## Continuous Deployment

To roll out an updated version of the Guestbook application, complete the following steps:

1. **Update the application manifests.** Make the desired changes in the `guestbook-demo` repository (for example, update the container image tag in the Deployment). Per-stage customizations can also be applied by editing the corresponding stage directory (e.g., `guestbook/staging`, `guestbook/canary`, `guestbook/prod`) in the `guestbook-demo` repository. Open a pull request, review, and merge to the main branch.
2. **Create a new release.** Tag a new release in the `guestbook-demo` repository to mark the updated version.
3. **Point to the new release.** In this repository, update `applications/simple-guestbook/simple-guestbook-application.yaml` to reference the new tag in the `spec.source.targetRevision` field.
4. **Add a new StagedUpdateRun.** Create a new `StagedUpdateRun` manifest (for example, `updateRun-2.yaml`) that references the same placement and update strategy with a unique name (e.g., `guestbook-release-v0.4.0`). Set `state: Initialize` to ensure you can examine the rollout details before it begins.
5. **Push the changes.** Commit and push to the `main` branch of this repository.
6. **Automatic reconciliation.** The Argo CD Application on the hub cluster detects the changes and triggers a new sync cycle.
7. **Review the rollout plan.** The new `StagedUpdateRun` in `Initialize` state generates a detailed rollout plan showing which clusters are assigned to each stage. Review the plan and resource snapshots to confirm you are rolling out what is intended and that the stage assignments and progression order are correct before proceeding.
8. **Approve and monitor.** Set the `StagedUpdateRun` to `state: Run` to start the rollout (either in the repo or via kubectl). Monitor progress and approve each stage as needed until the new version is fully deployed to all production clusters.
