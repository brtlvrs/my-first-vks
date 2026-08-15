# apps

Deploying an application into a namespace (Supervisor namespace directly, or a VKS workload
cluster). See [`../docs/07-repo-structure-and-conventions.md`](../docs/07-repo-structure-and-conventions.md)
for why this folder is component-first, unlike [`../platform/`](../platform/README.md).

<!-- toc -->

- [folder structure](#folder-structure)
- [which app is which](#which-app-is-which)
- [mise tasks](#mise-tasks)
- [adding a new app](#adding-a-new-app)
- [VM Service and vSphere Pods](#vm-service-and-vsphere-pods)
  * [when to reach for VM Service](#when-to-reach-for-vm-service)
  * [cloud-init bootstrap](#cloud-init-bootstrap)
  * [where apps get installed](#where-apps-get-installed)
  * [SSH keys](#ssh-keys)
  * [exposing ports: loadbalancer components](#exposing-ports-loadbalancer-components)
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
the Supervisor — copy whichever matches what you're building (see the table below).

## which app is which

| App | Kind | Runs on | Namespace resource in `base/`? | Notes |
|---|---|---|---|---|
| `hello-vks/` | Container | VKS guest cluster **or** Supervisor namespace directly | Yes | Restricted PSS, non-root (`nginx-unprivileged`) |
| `hello-vsphere-pod/` | Container, **vSphere Pod** (`runtimeClassName: runv`) | Supervisor namespace directly | No | Restricted PSS, non-root |
| `hello-vm/` | **VM Service** (`VirtualMachine` CR) | Supervisor namespace directly | No | Persistent data disk, cloud-init bootstrap, docker-compose demo — see [below](#vm-service-and-vsphere-pods) |
| `it-tools/` (`overlays/example-namespace-vks/`) | Container | VKS guest cluster | Yes (via a component) | Baseline PSS, root + `NET_BIND_SERVICE` only — see [`it-tools/README.md`](it-tools/README.md) |
| `it-tools/` (`overlays/example-namespace-vpod/`) | Container, **vSphere Pod** | Supervisor namespace directly | No | Same image and security posture as the VKS variant |

"Runs on: Supervisor namespace directly" means the app's Kubernetes namespace *is* the vSphere
Namespace itself — see [VM Service and vSphere Pods](#vm-service-and-vsphere-pods) below for why
that's structurally different from a normal VKS-guest-cluster app.

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
Pod. `it-tools/` applies the VKS-vs-vSphere-Pod split to one real app using
[kustomize components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)
instead of two separate app folders — see [`it-tools/README.md`](it-tools/README.md), including
why it can't use the Restricted Pod Security Standard the other examples do. See
[below](#when-to-reach-for-vm-service) for VM Service's clearest use case so far — vSphere Pods'
own is still open, see [`../TODO.md`](../TODO.md).

`hello-vm/` goes further, into a realistic rootless-Podman host with a persistent data disk — see
[`hello-vm/README.md`](hello-vm/README.md) for the full walkthrough. Five things about it worth
understanding up front:

### when to reach for VM Service

The clearest candidate use case so far: **a third-party vendor whose own support/operating model
is built around Podman/docker-compose and SSH access, not Kubernetes.** Asking them to
containerize for K8s compatibility is real work on their end, with no guarantee it happens the way
you'd want; a VM they can SSH into and manage exactly as they already know how meets them where
they are, while you still get to provision and govern it declaratively.

That last part is worth being precise about — "declarative" and "secure" get conflated easily. A
VM defined as a versioned `VirtualMachine` CR is more *reproducible* and *reviewable* than one
clicked together by hand: you can always redeploy from the same known spec instead of fixing drift
in place, and the bootstrap went through a PR before it ever ran, instead of a human following (or
half-following) a checklist. That's a real, useful property — but it is **not** the same claim as
"more secure." A badly-written `cloudConfig` is exactly as insecure declared as it would be built
by hand, just consistently so across every VM from that spec. `hello-vm/` itself is proof: it's
fully declarative, and it also exposes SSH via a LoadBalancer and falls back to password auth (see
[chapter 02](../docs/02-ssh-keys.md) and [below](#exposing-ports-loadbalancer-components)) — real
attack-surface choices that being declarative doesn't paper over. The honest framing: declarative
provisioning makes a VM's security posture *reviewable and consistent* — it's still on you to make
that posture *good*.

### cloud-init bootstrap

The whole VM gets configured via `base/vm.yaml`'s `spec.bootstrap.cloudInit.cloudConfig` — a
`users:` block (login user, sudo group, SSH key, a Secret-backed password) and a `runcmd:` list of
shell steps that run in order **the first time the VM boots**: format and mount the data disk,
install Podman and Docker Compose, and bind-mount the persistent paths (see below). **cloud-init
is one-shot** — re-applying an unchanged `VirtualMachine` spec over an already-running VM does
*not* re-run any of this, which is why changing something structural means a real redeploy
(`mise run vm:redeploy` — delete, then re-apply), not just `kubectl apply -k .` again.

### where apps get installed

`/home/hello` (or `/root`), not anywhere else on the VM. The OS disk itself is rebuilt from
scratch on every redeploy; only `/home/hello`, `/root`, and `/var/lib/hello-vm` directly are
bind-mounted from the separate, persistent data disk. Put your `docker-compose.yaml`, `.env`, and
any bind-mount targets it references under the home directory — see
[`hello-vm/README.md`](hello-vm/README.md#what-persists-across-a-redeploy-and-what-doesnt) for the
full persistence table.

### SSH keys

Added via cloud-init's native `ssh_authorized_keys` field, a plain list under `users[0]` in
`base/vm.yaml` (the `CHANGE_ME_SSH_PUBLIC_KEY` placeholder) — cloud-init writes them into
`~/.ssh/authorized_keys` itself at first boot, no manual step needed. See
[chapter 02](../docs/02-ssh-keys.md) for generating a key if you don't have one yet.

### exposing ports: loadbalancer components

vm-operator has its own `VirtualMachineService` CRD (a `LoadBalancer`-type resource, not a plain
`v1.Service` — a VM Service VM isn't a Pod) for external exposure. `hello-vm/` splits this into two
independent [kustomize components](https://kubectl.docs.kubernetes.io/guides/config_management/components/)
— `components/http-loadbalancer/` (the app's port) and `components/ssh-loadbalancer/` (port 22) —
so either can be toggled on or off in an overlay's `kustomization.yaml` `components:` list without
touching the other. That's the actual point of splitting them: you can expose the app publicly
while keeping SSH reachable only from inside the network, or vice versa, as two independent
decisions instead of one combined Service that couples them together. See
[`hello-vm/README.md`](hello-vm/README.md#reaching-it-from-outside-the-loadbalancer-components)
for the exact manifests and how to reach each one once deployed.

## CHANGE_ME placeholders in the example

| Placeholder | Where | Replace with |
|---|---|---|
| `example-namespace` (folder + `VCF_NAMESPACE`) | every `overlays/example-namespace/mise.toml` | your real vSphere Namespace |
| `service.beta.kubernetes.io/vsphere-load-balancer-class: "nsx"` | `hello-vks/base/service.yaml`, `hello-vsphere-pod/base/service.yaml` | your Supervisor's actual load-balancer class, if different |

`hello-vm/`'s own placeholders (VM class/image/storage class, SSH key) are documented in
[`hello-vm/README.md`](hello-vm/README.md#change_me-placeholders).
