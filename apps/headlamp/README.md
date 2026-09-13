# headlamp

A [Headlamp](https://headlamp.dev/) instance for this namespace's VKS guest cluster (or Supervisor
namespace, if deployed directly there) — a web UI for looking at the Deployments/Pods/Services
you `kubectl apply`, instead of only reading `kubectl get` output. Deploying this is a standard
step in [chapter 09](../../docs/09-deploying-your-first-app.md), not an optional example like the
other apps here.

Bound to the built-in, read-only `view` ClusterRole (see `base/clusterrolebinding.yaml`) —
upstream's own quickstart uses `cluster-admin`, deliberately scoped down here.

## accessing it

No `LoadBalancer` on purpose — reach it via `kubectl port-forward` and log in with a
`ServiceAccount` token:

```sh
kubectl create token headlamp -n headlamp --context <your-context>
kubectl port-forward -n headlamp svc/headlamp 4466:80 --context <your-context>
```

Then open <http://localhost:4466> and paste the token in as your login.

Tokens created this way are short-lived (1 hour by default) — just re-run `kubectl create token`
when one expires. This is the same tradeoff `kubectl proxy`/port-forward always has: convenient
for local use, nothing exposed on the network.
