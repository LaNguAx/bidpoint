# Bootstrap the local Kubernetes environment

## Purpose

Create the `bidpoint-local` Kind cluster and install Argo CD. This runbook stops after verifying Argo CD; registering the BidPoint repository and deploying workloads require the GitOps sources that follow this bootstrap.

This procedure implements [ADR 0003](../adr/0003.local-gitops-reconciliation.docs.md).

Review the [local environment foundations](../architecture/environment-foundations.docs.local.md) before running the bootstrap for the first time.

Run all commands from PowerShell. Docker Desktop must be running Linux containers.

## Pinned versions

| Component | Version |
| --- | --- |
| Kind CLI | `v0.33.0` |
| Kubernetes | `v1.36.4` |
| Kind node image | `kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed` |
| kubectl | `v1.37.0` |
| Helm | `v4.2.4` |
| Argo CD Helm chart | `10.4.0` |
| Argo CD application | `v3.5.1` |

Docker Desktop is a host prerequisite rather than a repository-managed runtime. The procedure is currently verified with Docker Engine `28.0.4`.

## 1. Verify prerequisites

```powershell
docker info
kind version
kubectl version --client
helm version
```

Do not continue unless Docker is reachable and the installed Kind, kubectl, and Helm versions match the pinned versions above.

Confirm that the target cluster does not already exist:

```powershell
$bidpointClusters = @(kind get clusters)
if ($bidpointClusters -contains 'bidpoint-local') {
    throw 'The bidpoint-local cluster already exists; stop instead of overwriting it.'
}
```

## 2. Create the cluster

```powershell
kind create cluster `
    --name bidpoint-local `
    --image 'kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed' `
    --wait 120s
```

Kind writes the cluster credentials to the default kubeconfig and selects the `kind-bidpoint-local` context.

## 3. Verify the cluster

Guard every cluster-changing command against the expected context:

```powershell
$bidpointContext = kubectl config current-context
if ($bidpointContext -ne 'kind-bidpoint-local') {
    throw "Expected kind-bidpoint-local, found $bidpointContext"
}
```

Verify that the control-plane node is ready and that the server uses the pinned Kubernetes version:

```powershell
kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes -o wide
kubectl version
kubectl cluster-info
```

## 4. Install Argo CD

Add and refresh the upstream chart repository:

```powershell
helm repo add argo https://argoproj.github.io/argo-helm --force-update
helm repo update argo
```

Recheck the context immediately before installation:

```powershell
$bidpointContext = kubectl config current-context
if ($bidpointContext -ne 'kind-bidpoint-local') {
    throw "Expected kind-bidpoint-local, found $bidpointContext"
}
```

Install the pinned chart with its local, non-high-availability defaults:

```powershell
helm upgrade --install argocd argo/argo-cd `
    --version 10.4.0 `
    --namespace argocd `
    --create-namespace `
    --wait `
    --timeout 10m `
    --atomic
```

`--atomic` rolls back a failed installation instead of leaving a partially installed Helm release.

## 5. Verify Argo CD

```powershell
helm status argocd --namespace argocd
kubectl wait --namespace argocd --for=condition=Ready pod --all --timeout=300s
kubectl get pods --namespace argocd
kubectl get crd applications.argoproj.io appprojects.argoproj.io applicationsets.argoproj.io
```

Bootstrap is complete when:

- the `bidpoint-local-control-plane` node is `Ready`;
- Helm reports the `argocd` release as `deployed`;
- every Argo CD Pod is `Running` and ready; and
- the Argo CD `Application`, `AppProject`, and `ApplicationSet` CRDs exist.

No BidPoint application is expected yet. Argo CD has not been given a repository or an `Application` resource at this stage.

## Troubleshooting

Preserve the full output of these commands before changing the cluster:

```powershell
helm status argocd --namespace argocd
kubectl get pods --namespace argocd -o wide
kubectl get events --namespace argocd --sort-by='.lastTimestamp'
```

If a Pod is not ready, inspect it by its exact name from `kubectl get pods`:

```powershell
kubectl describe pod --namespace argocd <pod-name>
kubectl logs --namespace argocd <pod-name> --all-containers
```

## Delete the local cluster

This permanently deletes the named local cluster and everything running in it. It does not delete repository files or Docker images.

```powershell
$bidpointContext = kubectl config current-context
if ($bidpointContext -ne 'kind-bidpoint-local') {
    throw "Expected kind-bidpoint-local, found $bidpointContext"
}

kind delete cluster --name bidpoint-local
```
