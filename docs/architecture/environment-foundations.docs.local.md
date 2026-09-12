# Local environment foundations and learning checkpoint

## Start the next session here

Tell Codex:

> Read `AGENTS.md` and `docs/architecture/environment-foundations.docs.local.md` completely, inspect the current working tree, and continue the local-environment learning session from the recorded checkpoint. Teach and brainstorm with me; explain each action before asking me to run it. Do not create the Kind cluster until I explicitly say I understand the creation step and am ready.

This document is both a durable explanation of the local platform and the checkpoint for its guided implementation.

## How to use this guide

The user is learning Docker, Kubernetes, Kind, Helm, Kustomize, Argo CD, GitOps, and Spring Cloud Config while implementing BidPoint.

- The user performs the implementation commands.
- Codex teaches, brainstorms, challenges assumptions, and reviews evidence.
- Work one checkpoint at a time.
- Explain what a command creates, changes, and verifies before presenting it.
- Do not jump from an explanation directly into a large sequence of commands.
- Preserve full failure output and diagnose it before retrying or deleting anything.

## Goal

Build the authoritative local integration environment:

```text
Windows
└── Docker Desktop
    └── Kind cluster: bidpoint-local
        └── Argo CD
            └── Spring Cloud Config Server
                └── serves one committed local configuration value
```

## Architecture decisions

- `local` runs in a Kind Kubernetes cluster.
- Argo CD runs inside the local cluster and reconciles that cluster.
- Cluster creation and initial Argo CD installation form the manual bootstrap boundary.
- BidPoint-owned workloads use Kustomize bases and environment overlays.
- Third-party platform software uses pinned upstream Helm charts.
- Argo CD reconciles resources rendered by both tools.
- The monorepo has one authoritative `main` branch and short-lived feature branches.
- Environment identity is represented by explicit config files and GitOps overlays, not long-lived environment branches.
- `gitops/` owns how workloads run.
- `config/` owns non-secret backend behavior.
- Spring Cloud Config serves `config/` from remote Git.
- Secret values never enter Git.
- Databases, messaging, ingress, and observability are deferred until an implemented workload needs them.

The decisions are recorded in [ADR 0003](../adr/0003.local-gitops-reconciliation.docs.md).

## Related implementation documents

- [ADR 0003: Reconcile the local Kubernetes environment with Argo CD](../adr/0003.local-gitops-reconciliation.docs.md)
- [Local Kubernetes bootstrap runbook](../runbooks/kubernetes-bootstrap.docs.local.md)

At the checkpoint recorded below, no cluster, Argo CD release, application, BidPoint container image, or remote resource has been created.

## Pinned local stack

| Component | Pin |
| --- | --- |
| Docker Engine used for validation | `28.0.4` |
| Kind CLI | `v0.33.0` |
| Kubernetes | `v1.36.4` |
| Kind node image | `kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed` |
| kubectl | `v1.37.0` |
| Kustomize bundled with kubectl | `v5.8.1` |
| Helm | `v4.2.4` |
| Argo CD Helm chart | `10.4.0` |
| Argo CD application | `v3.5.1` |

The Argo CD chart was downloaded into an isolated temporary Helm home and successfully rendered for Kubernetes `1.36.4`. The temporary directory was removed after validation.

## Workstation baseline

The user verified this baseline on 2026-08-29:

```text
Docker client/server: 28.0.4
Kind:                 v0.33.0
kubectl:              v1.37.0
Kustomize:            v5.8.1
Helm:                 v4.2.4
Kind clusters:        none
kubectl context:      not set
```

Windows previously selected Docker Desktop's bundled kubectl `v1.32.2`. The machine PATH was updated so the Winget kubectl `v1.37.0` now appears first. Docker's copy remains installed as a fallback.

## Practical checkpoint: before cluster creation

The user stopped immediately before cluster creation because the Kind command was presented before its effects were sufficiently understood.

The following has now been explained:

- Kind has no primary cluster-creation UI; its CLI is the supported interface.
- A Kind node is a Docker container configured to behave as a Kubernetes node.
- The first cluster will use one node that acts as both control plane and worker.
- Kind automates Docker networking, node creation, `kubeadm` initialization, certificates, core Kubernetes components, and kubeconfig generation.
- Cluster creation will add a Docker image, Docker container, Docker network, Kubernetes state inside the node, and an entry in `C:\Users\itaya\.kube\config`.
- It will not modify the BidPoint repository.
- Docker Desktop will show the resulting `bidpoint-local-control-plane` container.

Do not assume the user is ready to run the command merely because these facts are recorded. Briefly review the observable before-and-after state and ask whether the model is clear.

## Next action, only after the user is ready

The next command in the [local bootstrap runbook](../runbooks/kubernetes-bootstrap.docs.local.md) is:

```powershell
kind create cluster `
    --name bidpoint-local `
    --image 'kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed' `
    --wait 120s
```

Before running it, observe that no matching container or cluster exists:

```powershell
kind get clusters
docker ps --all --filter 'name=bidpoint-local'
```

After it succeeds, stop and inspect the created layers before installing Argo CD:

```powershell
kind get clusters
docker ps --filter 'name=bidpoint-local'
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods --all-namespaces
```

Expected context: `kind-bidpoint-local`. Expected node: `bidpoint-local-control-plane` in `Ready` state.

## Deferred decisions

- Where the Spring Cloud Config Server source belongs under `backend/`.
- Argo CD repository authentication if the GitHub repository is private.
- Config Server Git authentication.
- The repeatable local Secret-creation procedure.
- Whether Argo CD later manages upgrades of its own installation.
- Argo CD topology for `dev`, `stage`, and `prod`.
- Runtime dependencies required by the first real service.

## How the technologies build on each other

Study the stack in dependency order:

```text
Docker
  -> Kind
    -> Kubernetes objects and controllers
      -> Kustomize and Helm
        -> GitOps
          -> Argo CD
            -> Spring Cloud Config
```

### Docker

Docker runs isolated containers from immutable images. It is the container engine underneath the local environment. Kind asks Docker to run containers that behave as Kubernetes nodes.

Learn containers and images first. Kubernetes terminology is much easier once the difference between an image and a running container is clear.

### Kind

Kind means Kubernetes in Docker. It creates a conformant local Kubernetes cluster whose nodes are Docker containers. Kind owns cluster creation and deletion; it does not deploy BidPoint workloads.

### Kubernetes and kubectl

Kubernetes stores desired state as API objects and uses controllers to move actual state toward it. Pods run containers, Deployments manage replicated Pods, and Services give changing Pods stable network identities.

`kubectl` is a client for the Kubernetes API. It is not the cluster and does not keep workloads running after the command exits.

### Kustomize

Kustomize starts from complete Kubernetes YAML and produces variants by composing resources and applying overlays or patches. BidPoint uses it for first-party workloads because the repository needs explicit `local`, `dev`, `stage`, and `prod` differences rather than a reusable distribution package.

Study Kustomize only after raw Deployments and Services are readable; otherwise the base-and-overlay model lacks context.

### Helm

Helm is Kubernetes package management. A chart contains templates, default values, dependencies, and version metadata. BidPoint consumes pinned upstream charts for third-party platform software such as Argo CD.

When Argo CD uses a Helm source, Helm renders Kubernetes YAML and Argo CD owns synchronization. A separate `helm upgrade` process must not compete with Argo CD for the same resources.

### GitOps and Argo CD

GitOps makes Git the authoritative desired state. Argo CD runs inside Kubernetes, compares Git-rendered resources with live resources, reports drift, and can synchronize the cluster back to Git.

Argo CD does not create the cluster, build application images, or own application behavior. The initial Kind and Argo CD installation is the bootstrap boundary that must exist before reconciliation can begin.

### Spring Cloud Config

Spring Cloud Config serves non-secret backend behavior from remote Git. It is separate from GitOps: Argo CD reads `gitops/` to determine how software runs, while applications read `config/` through Config Server to determine how they behave.

Secrets belong in a secret-management system and enter workloads through Secret references; they do not belong in Git, Helm values, Kustomize overlays, or Config Server sources.

## Study path

Read the first group before resuming cluster creation. A quick overview of later technologies is fine, but focused study should wait until the prerequisite layer is visible in the running environment.

### Read before creating the Kind cluster

1. [Docker: What is a container?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)
2. [Docker: What is an image?](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)
3. [Kubernetes overview](https://kubernetes.io/docs/concepts/overview/)
4. [Kubernetes cluster architecture](https://kubernetes.io/docs/concepts/architecture/)
5. [Kubernetes controllers and desired state](https://kubernetes.io/docs/concepts/architecture/controller/)
6. [Kind initial design](https://kind.sigs.k8s.io/docs/design/initial/)
7. [Kind quick start](https://kind.sigs.k8s.io/docs/user/quick-start/)

### Read before creating workloads

These pages explain the raw resources that Kustomize will later combine. Understand the unmodified Kubernetes objects before studying their customization layer.

1. [Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
2. [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
3. [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
4. [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
5. [kubectl overview](https://kubernetes.io/docs/reference/kubectl/)
6. [Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)

### Read before installing and configuring Argo CD

At this stage, focus on the division of responsibility: Helm packages and renders third-party software, Kustomize builds first-party variants, and Argo CD reconciles the resulting Kubernetes resources from Git.

1. [OpenGitOps principles](https://opengitops.dev/)
2. [Argo CD architecture](https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/)
3. [Argo CD getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
4. [Helm introduction](https://helm.sh/docs/intro/introduction/)
5. [Argo CD Helm integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)

### Read before implementing Config Server

Read these after the deployment path is working. Config Server is an application-level configuration system, not a replacement for Kubernetes manifests or secret management.

1. [Spring Cloud Config overview](https://docs.spring.io/spring-cloud-config/reference/)
2. [Spring Cloud Config Server](https://docs.spring.io/spring-cloud-config/reference/server.html)
3. [Spring Cloud Config Client](https://docs.spring.io/spring-cloud-config/reference/client.html)
4. [Spring Cloud Config environment repositories](https://docs.spring.io/spring-cloud-config/reference/server/environment-repository.html)

## Resume verification

At the start of the next session:

1. Read repository `AGENTS.md` and relevant module READMEs.
2. Run `git status --short` and preserve all existing user changes.
3. Confirm `kind get clusters` still reports no clusters.
4. Confirm Docker Desktop is running.
5. Resume at the current learning checkpoint, not at Argo CD installation.
