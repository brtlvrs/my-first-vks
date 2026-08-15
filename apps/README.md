# apps

Deploying an application into a namespace (Supervisor namespace directly, or a VKS workload
cluster). See [`../docs/07-repo-structure-and-conventions.md`](../docs/07-repo-structure-and-conventions.md)
for why this folder is component-first, unlike [`../platform/`](../platform/README.md).

<!-- toc -->

- [folder structure](#folder-structure)
- [mise tasks](#mise-tasks)
- [adding a new app](#adding-a-new-app)
- [VM Service and vSphere Pods](#vm-service-and-vsphere-pods)
- [CHANGE_ME placeholders in the example](#change_me-placeholders-in-the-example)

<!-- tocstop -->

## folder structure

```
apps/
  <app>/
    mise.toml                    # optional: app-specific [tools] pins
    base/
      <app>.yaml (or split: namespace.yaml / deployment.yaml / service.yaml / ...)
      kustomization.yaml
    overlays/
      <vSphere Namespace>/
        mise.toml                 # sets VCF_NAMESPACE (and VCF_CLUSTER, if targeting a VKS cluster)
        kustomization.yaml
  mise-tasks/
    app.toml                      # app:render/apply/delete
    cluster.toml                  # cluster:status/nodes/pods-not-running/kubeconfig
    vm.toml                       # vm:power-on/power-off/restart/delete/redeploy/set-password
    whereamI.toml                 # wai — decode the current kube-context
```

Four fully worked examples live here, each demonstrating a different way to run something on
the Supervisor — copy whichever matches what you're building (see below):

- `hello-vks/` — a container, on a VKS guest cluster or directly in a Supervisor namespace
- `hello-vm/` — a standalone virtual machine, via VM Service
- `hello-vsphere-pod/` — a container, running natively on the Supervisor as a vSphere Pod
- `it-tools/` — a real public app (not a synthetic hello-world), deployed both ways at once —
  one shared `base/`, two overlays (VKS guest cluster and vSphere Pod), see
  [`it-tools/README.md`](it-tools/README.md)

## mise tasks

Run these from inside the specific `<app>/overlays/<namespace>/` folder you want to act on:

| Task | Does |
|------|------|
| `app:render` | `kustomize build .` — preview the rendered manifest |
| `app:apply` | `kubectl apply -k .` — deploy or update the app |
| `app:delete` | `kubectl delete -k .` — remove the app |
| `cluster:status` | Supervisor status if `VCF_CLUSTER` isn't set, else `vcf cluster get <cluster> --show-details` |
| `cluster:nodes` | `kubectl get nodes -o wide`, against the Supervisor or the specific VKS cluster |
| `cluster:pods-not-running` | `kubectl get pods -A --field-selector=status.phase!=Running` — quick cluster-wide health check |
| `cluster:kubeconfig` | Fetch the cluster's admin kubeconfig to `cluster-kubeconfig.yaml` |
| `wai` | "Where Am I" — prints Supervisor / vSphere Namespace / cluster / k8s namespace for your current context |

`vm:*` tasks (VM Service apps only, e.g. `hello-vm/`) are covered in
[`hello-vm/README.md`](hello-vm/README.md#vm-mise-tasks).

Also see the root [`mise-tasks/context.toml`](../mise-tasks/context.toml) tasks for authenticating
before any of the above will work.

## adding a new app

1. `cp -r apps/hello-vks apps/<new-app>` as a starting point.
2. Edit `<new-app>/base/*.yaml` — at minimum rename the `Namespace`, `Deployment`, and `Service`,
   and swap the container image.
3. In `<new-app>/overlays/<namespace>/kustomization.yaml`, set `namespace:` to the app's own
   Kubernetes namespace name (this can be different from the vSphere Namespace — see
   `docs/07-repo-structure-and-conventions.md`).
4. In `<new-app>/overlays/<namespace>/mise.toml`, set `VCF_NAMESPACE` to the target vSphere
   Namespace (and `VCF_CLUSTER` if you're deploying onto a specific VKS cluster rather than
   directly into a Supervisor namespace).
5. From inside that folder: `mise run app:render` to sanity-check the output, then
   `mise run context:supervisor` (or `context:use`/`context:cluster`, if already logged in) and
   `mise run app:apply`.
6. To deploy the same app into a second namespace, `cp -r overlays/<namespace> overlays/<other-namespace>`
   and repeat steps 3-5 for the copy.

## VM Service and vSphere Pods

`hello-vm/` and `hello-vsphere-pod/` both deploy directly into the vSphere Namespace itself,
never onto a VKS guest cluster — see [chapter 00](../docs/00-introduction.md) for what each
capability is. Two structural differences from `hello-vks/` worth noticing if you're copying
either as a starting point:

- No `Namespace` resource in `base/` — the target namespace *is* the vSphere Namespace (set via
  the overlay's `namespace:` field), not an app-specific one you create yourself.
- Their overlay `mise.toml` sets `VCF_NAMESPACE` only, never `VCF_CLUSTER` — there's no guest
  cluster in the picture for either of these.

`hello-vsphere-pod/` is otherwise the same restricted-PSS-compliant deployment as `hello-vks/`,
plus `runtimeClassName: runv` in the pod spec — that field is what actually makes it a vSphere
Pod. `hello-vm/` goes further, into a realistic rootless-Podman host with a persistent data disk
— see [`hello-vm/README.md`](hello-vm/README.md) for its deploy steps, what survives a redeploy,
and its own `vm:*` mise tasks. `it-tools/` applies the VKS-vs-vSphere-Pod split to one real app
using [kustomize components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)
instead of two separate app folders — see [`it-tools/README.md`](it-tools/README.md), including
why it can't use the Restricted Pod Security Standard the other examples do. Concrete guidance on
when to reach for VM Service or vSphere Pods over a regular `hello-vks`-style app is still being
worked out — see [`../TODO.md`](../TODO.md).

## CHANGE_ME placeholders in the example

| Placeholder | Where | Replace with |
|---|---|---|
| `example-namespace` (folder + `VCF_NAMESPACE`) | every `overlays/example-namespace/mise.toml` | your real vSphere Namespace |
| `service.beta.kubernetes.io/vsphere-load-balancer-class: "nsx"` | `hello-vks/base/service.yaml`, `hello-vsphere-pod/base/service.yaml` | your Supervisor's actual load-balancer class, if different |

`hello-vm/`'s own placeholders (VM class/image/storage class, SSH key) are documented in
[`hello-vm/README.md`](hello-vm/README.md#change_me-placeholders).
