# platform

Cluster lifecycle: provisioning VKS clusters and enabling per-namespace Velero backups on the
Supervisor. See [`../docs/07-repo-structure-and-conventions.md`](../docs/07-repo-structure-and-conventions.md)
for why this folder is laid out differently from [`../apps/`](../apps/README.md).

<!-- toc -->

- [folder structure](#folder-structure)
- [mise tasks](#mise-tasks)
- [adding a new namespace / cluster](#adding-a-new-namespace--cluster)
- [CHANGE_ME placeholders in the example](#change_me-placeholders-in-the-example)

<!-- tocstop -->

## folder structure

```
platform/
  bases/                       # one kustomize base per service type, shared by every namespace
    vks-cluster/                # default VKS workload cluster definition
      cluster.yaml
      kustomization.yaml
    velero/                     # default per-namespace backup config
      veleroservice.yaml
      kustomization.yaml
  <vSphere Namespace>/          # one folder per Supervisor namespace — services live as siblings
    mise.toml                   # sets VCF_NAMESPACE, inherited by every service overlay below
    <vks-cluster name>/         # overlay: this namespace's cluster instance
      mise.toml                 # sets VCF_CLUSTER
      kustomization.yml
    velero/                     # overlay: this namespace's Velero backup config (no VCF_CLUSTER needed)
      kustomization.yml
  mise-tasks/
    cluster.toml                # cluster:render/diff/apply/status/nodes/delete/kubeconfig
```

`bases/*/` are never applied directly — every field that matters is a `CHANGE_ME_*` placeholder,
overridden by an overlay's JSON6902 patch. `example-namespace/` is a fully worked example built
on top of them — copy it as your starting point (see below) rather than starting from `bases/`.

## mise tasks

Run these from inside the specific `<namespace>/<service>/` folder you want to act on —
`VCF_NAMESPACE`/`VCF_CLUSTER` come from that folder's (and its parents') `mise.toml`:

| Task | Does |
|------|------|
| `cluster:render` | `kustomize build .` — preview the rendered manifest |
| `cluster:diff` | `kubectl diff -k .` against the live cluster — preview a change before applying it |
| `cluster:apply` | `kubectl apply -k .` — create the resource, or push a spec change to an existing one |
| `cluster:status` | Supervisor status if `VCF_CLUSTER` isn't set, else `vcf cluster get <cluster> --show-details` |
| `cluster:nodes` | `kubectl get nodes -o wide`, against the Supervisor or the specific VKS cluster |
| `cluster:kubeconfig` | Fetch the cluster's admin kubeconfig to `cluster-kubeconfig.yaml` |
| `cluster:delete` | **Destructive.** Interactive (`fzf` picker, or pass a name), asks you to type the name to confirm |

Also see the root [`mise-tasks/context.toml`](../mise-tasks/context.toml) tasks (`context:supervisor`,
`context:cluster`, `context:use`, `context:refresh`) for authenticating before any of the above will work.

## adding a new namespace / cluster

1. `cp -r platform/example-namespace platform/<new-namespace>` as a starting point.
2. In `<new-namespace>/mise.toml`, set `VCF_NAMESPACE` to the real vSphere Namespace name.
3. In `<new-namespace>/<cluster-folder>/mise.toml`, set `VCF_CLUSTER` to the real cluster name
   (you can rename the folder itself too — the folder name doesn't have to match the value).
4. In `<new-namespace>/<cluster-folder>/kustomization.yml`, update the JSON6902 patch values —
   at minimum `/metadata/name`, the `vmClass`/`storageClass` variable values, and the replica
   counts. Keep `resources:` pointing at `../../bases/vks-cluster` and `namespace:` set to the
   target vSphere Namespace.
5. From inside that folder: `mise run cluster:render` to sanity-check the output, then
   `mise run context:supervisor` (or `context:use`, if already logged in) and `mise run cluster:apply`.
6. Repeat for `<new-namespace>/velero/` if this namespace needs backups — see
   [`../docs/11-backups-with-velero.md`](../docs/11-backups-with-velero.md).

A version bump, a scale change, or any other spec edit is just a normal patch-value change
followed by `cluster:diff` then `cluster:apply` — there's no separate "upgrade" task.
**Keep the control-plane replica count odd** (1, 3, 5) — etcd needs a majority quorum.

## CHANGE_ME placeholders in the example

| Placeholder | Where | Replace with |
|---|---|---|
| `example-namespace` (folder + `VCF_NAMESPACE`) | `example-namespace/mise.toml`, both `kustomization.yml` `namespace:` fields | your real vSphere Namespace |
| `example-cluster` (folder + `VCF_CLUSTER` + patch value) | `vks-cluster/mise.toml`, `vks-cluster/kustomization.yml` | your real cluster name |
| `example-storage-policy` | `vks-cluster/kustomization.yml` | a storage policy name from `kubectl get storageclass` |
| `example-bucket`, `us-east-1`, `minio.example.internal:9000` | `velero/kustomization.yml` | your real S3-compatible bucket/region/endpoint — see [`../docs/03-required-platform-services.md`](../docs/03-required-platform-services.md) |
