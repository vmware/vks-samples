# GitOps for VKS clusters, VMs and apps with Argo CD on VCF 9.1.1

One Git repo drives everything: VKS clusters, VMs, and the apps running on them.
Any VKS cluster labelled `env: dev` or `env: prod` is attached to Argo CD automatically
and receives the matching app.

## How it's organised

All paths are relative to `gitops/argocd-auto-attach/`.

| Path | What it is |
|---|---|
| `bootstrap-argocd-app.yaml` | Bootstrap Application in Argo CD. The only file you apply by hand. It points Argo CD at `argocd/` folder |
| `argocd/` | Define Argo CD Application for infra provisioning and ApplicationSet for containerized app deployment |
| `infra/platform/` | Platform setup: RBAC and the policies that auto-attach any VKS cluster labelled `env: dev` or `env: prod` |
| `infra/<env>/` | One folder per environment: a VKS cluster and a VM |
| `apps/demo-app/<env>/` | What runs on each environment's VKS cluster |

**The rule:** a cluster labelled `env: prod` is attached to Argo CD automatically and gets `apps/demo-app/prod` app deployed on it. Similar logic for `env:dev`.

## Prerequisites

- A VCF 9.1.1 environment with a vSphere Supervisor and the Argo CD 1.2.0 service enabled
- A vSphere Namespace (the "Supervisor namespace") with an Argo CD instance running in it
- That namespace has a VM class, a storage class and a VM image assigned (and a zone, if you use zones)
- `kubectl` access to the Supervisor namespace
- **Your own fork of this repository.** Argo CD syncs from Git, so it has to read from a repo
  you can push your lab-specific edits to.

## Setup

### 1. Fork and clone

Fork `vmware/vks-samples`, clone your fork, and go to `gitops/argocd-auto-attach/`.

### 2. Edit the files for your lab

Every line you may need to change is marked with a `CHANGE ME` comment. Search for them with:

```sh
grep -rn "CHANGE ME" .
```

| What | Where | Notes |
|---|---|---|
| Supervisor namespace (`my-ns` in the samples) | Every YAML file in `bootstrap-argocd-app.yaml`, `argocd/`, `infra/` | Replace all occurrences with your namespace |
| Argo CD cluster name for the Supervisor (`destination.name`) | `bootstrap-argocd-app.yaml`, `argocd/infra-argocd-app.yaml` | See it with `argocd cluster list` |
| Git repo URL (`repoURL`) | `bootstrap-argocd-app.yaml`, `argocd/infra-argocd-app.yaml`, `argocd/demo-app-argocd-applicationset.yaml` | Point at your fork |
| Git branch (`targetRevision`) | same three files | Default is `main` |
| ClusterClass, Kubernetes version, VM class, storage class | `infra/dev/vks-cluster.yaml` | Must exist in your environment |
| VM class, VM image (`vmi-...`), storage class, zone | `infra/dev/vm.yaml` | Image IDs differ per content library |

Do **not** change the `env: dev` / `env: prod` labels or the `env` keys in
`infra/platform/auto-attach-policies.yaml`. The auto-attach behavior depends on them.

Commit and push your edits to your fork.

### 3. Bootstrap

Run this command in the vSphere namespace where Argo CD instance is running:

```sh
kubectl apply -f bootstrap-argocd-app.yaml
```

Argo CD then creates the `infra` Application and the `demo-app` ApplicationSet:

1. `infra/platform/` is applied first (sync wave `-1`): RBAC and the auto-attach policies.
2. Argo CD creates the dev VKS cluster and the dev VM.
3. The cluster attaches itself to Argo CD with the label `env=dev`.
4. The ApplicationSet sees the attached cluster and deploys `apps/demo-app/dev` to it.

### 4. Check the result

```sh
kubectl get cluster,virtualmachine,virtualmachineservice -n <namespace>
```

The VM and the app are both exposed through a LoadBalancer. Open their external IPs in a
browser to see the "ENV: DEV" pages.

## Add a new environment

This sample ships with `infra/dev/` only. To add `prod`:

1. Copy `infra/dev/` to `infra/prod/`.
2. In the copied files, rename `dev` to `prod`: the cluster and VM names, the `env` labels,
   and the cloud-init Secret and VM selector names. Adjust sizes if you like.
3. Open a pull request. After it's merged:
   1. Argo CD creates the VKS cluster and the VM.
   2. The cluster attaches itself to Argo CD with `env=prod`.
   3. Argo CD deploys `apps/demo-app/prod` to it.

## Promote a change

Change `apps/demo-app/dev/app.yaml` and check it in dev, then open a pull request making the
same change in `apps/demo-app/prod/app.yaml`.

## Clean up

Deleting the `infra` Application deletes the VKS clusters and VMs it created (it has a
cascade-delete finalizer). Delete the `root` Application to remove everything.
