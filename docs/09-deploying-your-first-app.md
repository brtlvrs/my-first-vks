# 09 — deploying your first app

Walks through `apps/hello-vks/` end to end. See [`apps/README.md`](../apps/README.md) for the
full task reference.

<!-- toc -->

- [1. copy the example](#1-copy-the-example)
- [2. point it at your namespace/cluster](#2-point-it-at-your-namespacecluster)
- [3. authenticate](#3-authenticate)
- [4. render, then apply](#4-render-then-apply)
- [5. verify](#5-verify)

<!-- tocstop -->

## 1. copy the example

```
cp -r apps/hello-vks apps/<your-app>
```

For your very first deploy, it's fine to skip the copy and just deploy `hello-vks` as-is —
everything below assumes you did, but the steps are identical either way.

## 2. point it at your namespace/cluster

Edit `overlays/example-namespace/mise.toml`:

```toml
[env]
VCF_NAMESPACE = "<your-namespace>"
# only if deploying onto a specific VKS cluster rather than directly into a Supervisor namespace:
VCF_CLUSTER = "<your-cluster>"
```

Rename the `overlays/example-namespace/` folder to match, if you like — same rule as
[chapter 08](08-provisioning-a-cluster.md#2-fill-in-the-placeholders), the folder name is just for
navigation.

## 3. authenticate

```
cd apps/hello-vks/overlays/example-namespace
mise run context:use
```

(`context:use` picks up whatever context you already created for this namespace/cluster in
[chapter 06](06-connecting-to-supervisor.md) or [chapter 08](08-provisioning-a-cluster.md); use
`context:cluster` instead if this is the first time targeting this specific VKS cluster.)

## 4. render, then apply

```
mise run app:render
```

Check the output, then:

```
mise run app:apply
```

## 5. verify

```
mise run wai
kubectl get pods -n hello-vks
kubectl get svc -n hello-vks
```

`wai` ("where am I") confirms which Supervisor/namespace/cluster context you're actually pointed
at — useful the first several times, until reading `kubectl config current-context` directly
becomes second nature. The `Service` is `type: LoadBalancer` — once your Supervisor's load
balancer provisions an external IP (`kubectl get svc -n hello-vks` will show it under
`EXTERNAL-IP`, may take a minute), open it in a browser: you should see the default nginx welcome
page.

To remove it again: `mise run app:delete` from the same folder. To learn how to check on it going
forward (health, node status, scaling), continue to
[chapter 10](10-day2-operations.md).
