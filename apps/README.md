# apps

Deploying an application into a namespace (Supervisor namespace directly, or a VKS workload
cluster). See [`../docs/07-repo-structure-and-conventions.md`](../docs/07-repo-structure-and-conventions.md)
for why this folder is component-first, unlike [`../platform/`](../platform/README.md).

<!-- toc -->

- [folder structure](#folder-structure)
- [mise tasks](#mise-tasks)
- [adding a new app](#adding-a-new-app)
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
    whereamI.toml                 # wai — decode the current kube-context
```

`hello-vks/` is a fully worked example — copy it as your starting point (see below).

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

## CHANGE_ME placeholders in the example

| Placeholder | Where | Replace with |
|---|---|---|
| `example-namespace` (folder + `VCF_NAMESPACE`) | `overlays/example-namespace/mise.toml` | your real vSphere Namespace |
| `service.beta.kubernetes.io/vsphere-load-balancer-class: "nsx"` | `base/service.yaml` | your Supervisor's actual load-balancer class, if different |
