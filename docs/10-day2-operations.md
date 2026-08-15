# 10 — day-2 operations

Everyday operational tasks once a cluster and its apps are up.

<!-- toc -->

- [status and node health](#status-and-node-health)
- [fetching a kubeconfig](#fetching-a-kubeconfig)
- [scaling a cluster](#scaling-a-cluster)
- [upgrading a cluster's Kubernetes version](#upgrading-a-clusters-kubernetes-version)
- [deleting a cluster](#deleting-a-cluster)
- [accessing a node's shell without SSH](#accessing-a-nodes-shell-without-ssh)

<!-- tocstop -->

## status and node health

From inside any `platform/<namespace>/<service>/` or `apps/<app>/overlays/<namespace>/` folder:

```
mise run cluster:status
mise run cluster:nodes
```

Both fall back to Supervisor-level status/nodes if `VCF_CLUSTER` isn't set for that folder,
otherwise they target the specific VKS cluster. From `apps/`, there's also a cluster-wide health
check across every namespace:

```
mise run cluster:pods-not-running
```

## fetching a kubeconfig

```
mise run cluster:kubeconfig
```

Writes `cluster-kubeconfig.yaml` into your current folder (gitignored — never commit this). Use
it for tools that expect a plain kubeconfig file rather than a `vcf`-managed context.

## scaling a cluster

There's no dedicated "scale" task — it's a normal declarative change like any other:

1. Edit the relevant replica count in `platform/<namespace>/<cluster>/kustomization.yml`:
   - **Workers**: `/spec/topology/workers/machineDeployments/0/replicas`. Safe to change in
     either direction.
   - **Control plane**: `/spec/topology/controlPlane/replicas`. **Keep this odd** (1, 3, 5) — etcd
     needs a majority quorum; an even number adds a node that can't break a tie without improving
     fault tolerance. Scaling down is riskier than up — confirm `cluster:status` reports every
     existing control-plane node healthy *before* you reduce this.
2. `mise run cluster:diff` to preview, then `mise run cluster:apply`.

## upgrading a cluster's Kubernetes version

Same mechanism — a version bump is just another field:

1. `kubectl get clusterclass builtin-generic-v3.6.0 -n vmware-system-vks-public -o yaml` lists
   which versions the ClusterClass actually supports.
2. Edit `/spec/topology/version` in the cluster's `kustomization.yml` to the new version.
3. `mise run cluster:diff`, then `mise run cluster:apply`.
4. This triggers a **rolling** upgrade — control-plane nodes one at a time (to preserve etcd
   quorum), then worker nodes. Watch it with `mise run cluster:nodes` / `cluster:status` until
   every node reports the new version. Treat your first upgrade on any cluster that matters as a
   dry run: check status immediately before and after, and confirm your apps survive the rolling
   restart.

## deleting a cluster

```
mise run cluster:delete
```

**Destructive.** Interactively picks a cluster (via `fzf`, if installed — see
[`TODO.md`](../TODO.md)) or accepts a name directly (`mise run cluster:delete <name>`), then
requires you to type the cluster's name again to confirm before anything happens.

## accessing a node's shell without SSH

If you have working `kubectl` access to the cluster itself (i.e. `mise run context:cluster`/
`context:use` already succeeded), you generally don't need SSH to reach a node's shell:

```
kubectl debug node/<node-name> -it --image=busybox -n kube-system --context <namespace>:<cluster-name>
```

Run this **in `kube-system`**, not an application namespace — namespaces enforcing the
`restricted` Pod Security Standard (like `hello-vks`, and like any namespace you provision
following [chapter 09](09-deploying-your-first-app.md)) will refuse to schedule the privileged
debug pod this needs. This drops you into a container with the node's root filesystem mounted at
`/host`, sharing the node's PID/network namespaces:

```
chroot /host
```

gives a real root shell on the node, with its actual `ss`/`crictl`/`journalctl` etc. — not
busybox's limited versions.

**Clean up afterward** — `kubectl debug node` doesn't delete its pod automatically:

```
kubectl get pod -n kube-system --context <namespace>:<cluster-name> | grep node-debugger
kubectl delete pod <that-pod-name> -n kube-system --context <namespace>:<cluster-name>
```

If the workload-cluster API itself is down (so there's no `kubectl debug node` path in at all),
VKS auto-generates a `<cluster-name>-ssh` keypair secret and a `<cluster-name>-ssh-password`
secret (key `ssh-passwordkey`) in the Supervisor namespace as a fallback, login user
`vmware-system-user` — or use the vCenter Web UI's "Launch Web Console" button for
network-independent console access (no SSH needed at all, but also no copy-paste).
