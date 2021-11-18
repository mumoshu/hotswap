# Okra

`Okra` is a [Kubernetes controller](https://kubernetes.io/docs/concepts/architecture/controller/) and a set of [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) which provide advanced multi-cluster appilcation rollout capabilities, such as canary deployment of clusters.

`okra` eases managing a lot of ephemeral Kubernetes clusters.

If you've been using ephemeral Kubernetes clusters and employed blue-green or canary deployments for zero-downtime cluster updates, you might have suffered from a lot of manual steps required. `okra` is intended to automate all those steps.

In a standard scenario, a system update with `okra` would like the below.

- **You** provision one or more new clusters with cluster tags like `name=web-1-v2, role=web, version=v2`
- **Okra** auto-imports the clusters into **ArgoCD**
- **ArgoCD ApplicationSet** deploys your apps onto the new clusters
- **Okra** updates the loadbalancer configuration to gradually migrate traffic to the new clusters, while running various checks to ensure application availability

## Project Status and Scope

`okra` (currently) integrates with AWS ALB and NLB and target groups for traffic management, CloudWatch Metrics and Datadog for canary analysis.

`okra` currently works on AWS only, but the design and the implementation of it is generic enough to be capable of adding more IaaS supports. Any contribution around that is welcomed.

Here's the list of possible additional IaaSes that the original author (@mumoshu) has thought of:

- Cluster API
- GKE

Here's the list of possible additional loadbalancers:

- Envoy
- [Istio Ingerss Gateway](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/)
- [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)

## How it works

`Okra` manages **cells** for you. A cell can be compared to a few things.

A cell is like a Kubernetes pod of containers. A Kubernetes pod an isolated set of containers, where each container usually runs a single application, and you can have two or more pods for availability and scalability. A Okra [`cell`](/crd.md#cell) is a set of Kubernetes clusters, where each cluster runs your application and you can have two or more clusters behind a loadbalancer for horizontal scalability beyond the limit of a single cluster.

A cell is like a storage array but for Kubernetes clusters. You hot-swap a disk in a storage array while running. Similarly, with `okra` you hot-swap a cluster in a cell while keeping your application up and running.

Okra's `cell-contorller` is responsible for managing the traffic shift across clusters.

You give each [`Cell`](/crd.md#cell) a set of settings to discover AWS target groups and configure loadbalancers, and metrics.

The controller periodically discovers AWS target groups. Once there are enough number of new target groups, it then compares the target groups associated to the loadbalancer. If there's any difference, it starts updating the ALB while checking various metrics for safe rollout.

Okra uses Kubernetes CRDs and custom resources as a state store and uses the standard Kubernetes API to interact with resources.

Okra calls various AWS APIs to create and update AWS target groups and update AWS ALB and NLB forward config for traffic management.

### Comparison with Flagger and Argo Rollouts

Unlike `Argo Rollouts` and `Flagger`, in `Okra` there is no notions of "active" and "preview" services for a blue-green deployment, or "canary" and "stable" services for a canary deployment.

It assumes there's one or more target groups per cell. `cell` basically does a canary deployment, where the old set of target groups is consdidered "stable" and the new set of target groups is considered "canary".

In `Flagger` or `Argo Rollouts`, you need to update its K8s resource to trigger a new rollout. In Okra you don't need to do so. You preconfigure its resource and Okra auto-starts a rollout once it discovers enough number of new target groups.

## Concepts

`okra` updates your [`Cell`](/crd.md#cell).

A okra `Cell` is composed of target groups and an AWS loadbalancer, and a set of metrics for canary anlysis.

Each target group is tied to a `cluster`, where a `cluster` is a Kubernetes cluster that runs your container workloads.

An `application` is deployed onto `clusters` by `ArgoCD`. The traffic to the `application` is routed via an [AWS ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) in front of `clusters`.

```
kind: Cell
spec:
  ingress:
    type: AWSApplicationLoadBalancer
    awsApplicationLoadBalancer:
      listenerARN: ...
      targetGroupSelector:
        matchLabels:
          role: web
  replicas: 2
  versionedBy:
    label: "okra.mumoshu.github.io/version"
```

`okra` acts as an application traffic migrator.

It detects new `target groups`, and live migrate traffic by hot-swaping old target groups serving the affected `applications` with the new target groups, while keepining the `applications` up and running.

## Usage

It is inteded to be deployed onto a "control-plane" cluster to where you usually deploy applications like ArgoCD.

It requires you to use:

- NLB or ALB to load-balance traffic "across" clusters
  - You bring your own LB, Listener, and tell `okra` the Listener ID, Number of Target Groups per Cell, and a label to group target groups by version.
- Uses ArgoCD ApplicationSets to deploy your applications onto cluster(s)

In the future, it may add support for using Route 53 Weighted Routing instead of ALB.

Although we assume you use ApplicationSet for app deployments, it isn't really a strict requirement. Okra doesn't communiate with ArgoCD or ApplicationSet. All Okra does is to discover EKS clusters, create and label target groups for the discovered clusters, and rollout the target groups. You can just bring your own tool to deploy apps onto the clusters today.

It supports complex configurations like below:

- One or more clusters per cell, or an ALB listener rule. Imagine a case that you need a pair of clusters to serve your service. `okra` is able to canary-deploy the pair of clusters, by periodically updating two target group weights as a whole.

## CRDs

`Okra` provides several Kuberntetes CustomResourceDefinitions(CRD) to achieve its goal.

See [crd.md](crd.md) for more documentation and details of each CRD.

## CLI

`okra` is the CLI application that consists of the controller and other utility commands for testing.

See [CLI](/cli.md) for more information and its usage.

## Related Projects

Okra is inspired by various open-source projects listed below.

- [ArgoCD](https://argoproj.github.io/argo-cd/) is a continuous deployment system that embraces GitOps to sync desired state stored in Git with the Kubernetes cluster's state. `okra` integrates with `ArgoCD` and especially its `ApplicationSet` controller for applicaation deployments.
  - `okra` relies on ArgoCD `ApplicationSet` controller's [`Cluster Generator` feature](https://argocd-applicationset.readthedocs.io/en/stable/Generators/#label-selector)
- [Flagger](https://flagger.app/) and [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) enables canary deployments of apps running across pods. `okra` enables canary deployments of clusters running on IaaS.
- [argocd-clusterset](https://github.com/mumoshu/argocd-clusterset) auto-discovers EKS clusters and turns those into ArgoCD cluster secrets. `okra` does the same with its [`ClusterSet` CRD](/crd.md#clusterset) and `argocdcluster-controller`.
- [terraform-provider-eksctl's courier_alb resource](https://github.com/mumoshu/terraform-provider-eksctl/tree/master/pkg/courier) enables canary deployments on target groups behind AWS ALB with metrics analysis for Datadog and CloudWatc metrics. `okra` does the same with it's [`AWSApplicationLoadBalancerConfig` CRD](/crd.md#awsapplicationloadbalancerconfig) and `awsapplicationloadbalancerconfig-controller`.

## Why is it named "okra"?

Initially it was named `kubearray`, but the original author wanted something more catchy and pretty.

In the beginning of this project, the author thought that hot-swapping a cluster while keeping your apps running looks like hot-swaping a drive while keeping a server running.

We tend to call a cluster of storages where each storage drive can be hot-swapped a "storage array", hence calling a tool to build a cluster of clusters where each cluster can be hot-swapped "kubearray" seemed like a good idea.

Later, he searched over the Internet for a prettier and catchier alternative. While browsing a list of cool Japanese terms with 3 syllables, he encountered "okra". "Okra" is a pod vegetable full of edible seeds. The term is relatively unique that it sounds almost the same in both Japanese and English. The author thought that "okra" can be a good metaphor for a cluster of sub-clusters when each seed in an okra is compared to a sub-cluster.
