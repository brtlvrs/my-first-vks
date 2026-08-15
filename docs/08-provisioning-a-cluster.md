# 08 — provisioning a cluster

Walks through `platform/example-namespace/` end to end. See
[`platform/README.md`](../platform/README.md) for the full task reference and placeholder table
this chapter is summarizing.

<!-- toc -->

- [1. copy the example](#1-copy-the-example)
- [2. fill in the placeholders](#2-fill-in-the-placeholders)
- [3. authenticate](#3-authenticate)
- [4. render, then apply](#4-render-then-apply)
- [5. watch it come up](#5-watch-it-come-up)

<!-- tocstop -->

## 1. copy the example

```
cp -r platform/example-namespace platform/<your-namespace>
```

Use the real vSphere Namespace name you were assigned (see
[chapter 06](06-connecting-to-supervisor.md#roles-and-permissions)).

## 2. fill in the placeholders

- `<your-namespace>/mise.toml` — set `VCF_NAMESPACE` to the real namespace name.
- `<your-namespace>/vks-cluster/mise.toml` — set `VCF_CLUSTER` to your chosen cluster name.
- `<your-namespace>/vks-cluster/kustomization.yml` — update the JSON6902 patch values:
  `/metadata/name` (cluster name, matching `VCF_CLUSTER` above), the `vmClass`/`storageClass`
  variable values (`kubectl get vmclass` / `kubectl get storageclass` list what's actually
  available in your namespace once logged in), and the replica counts. Also update `namespace:`
  at the top of the file to match your real namespace.

You can rename the `vks-cluster/` folder itself too — the folder name is just for navigation, the
actual cluster name comes from the patch value and `VCF_CLUSTER`.

## 3. authenticate

```
cd platform/<your-namespace>/vks-cluster
mise run context:supervisor
```

(Skip this if you've already authenticated against this Supervisor from elsewhere in the repo —
see [chapter 06](06-connecting-to-supervisor.md).)

## 4. render, then apply

```
mise run cluster:render
```

Read the output — this is exactly what will be sent to the API. Once it looks right:

```
mise run cluster:apply
```

## 5. watch it come up

```
mise run cluster:status
mise run cluster:nodes
```

Cluster provisioning takes several minutes — control-plane nodes come up first, then workers.
`cluster:status` and `cluster:nodes` are safe to re-run as often as you like while you wait.

Once it's ready, continue to [chapter 09](09-deploying-your-first-app.md) to deploy something
onto it, or [chapter 11](11-backups-with-velero.md) to enable backups for this namespace.
