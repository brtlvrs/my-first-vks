# 10 — day-2 operations

Everyday operational tasks once a cluster and its apps are up.

<!-- toc -->

- [status and node health](#status-and-node-health)
- [Pod Security Standards: understanding admission](#pod-security-standards-understanding-admission)
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

## Pod Security Standards: understanding admission

Every namespace `apps/` deploys into has a **Pod Security Standard** enforced on it — a
Kubernetes-native admission check (built into the API server since 1.25, nothing extra to
install) that rejects a pod outright at `kubectl apply` time if it doesn't comply, before it's
even scheduled. Worth understanding on day 2, because it's a common source of "why won't this
even create" confusion that looks different from every other kind of failure — see
[chapter 12](12-troubleshooting-cookbook.md#app-pod-stuck-pending-or-refused-by-admission) for how
to recognize it when it happens.

It's turned on per-namespace with a label — you've already seen it in every app's `base/`:

```yaml
pod-security.kubernetes.io/enforce: restricted   # or baseline, or privileged
pod-security.kubernetes.io/enforce-version: latest
```

Three levels, each stricter than the last:

- **Privileged** — no restrictions at all.
- **Baseline** — blocks the obviously dangerous stuff: privileged containers, host
  networking/PID/IPC, most `hostPath` mounts, capability *additions* beyond a small allowlist.
  Does **not** require running as non-root or dropping capabilities.
- **Restricted** — everything Baseline blocks, plus it *requires*, at pod or container level:
  - `runAsNonRoot: true`
  - `allowPrivilegeEscalation: false`
  - `seccompProfile.type: RuntimeDefault` (or `Localhost`)
  - capabilities: must `drop: ["ALL"]`, and may only add back `NET_BIND_SERVICE`

Two worked examples in this repo show both ends of this:

- `apps/hello-vks/` (and `hello-vsphere-pod/`) use `nginxinc/nginx-unprivileged` specifically
  because it satisfies all four Restricted rules cleanly — it's built to run as UID 1000 on port
  8080, so `runAsNonRoot: true` costs nothing.
- `apps/it-tools/` is the more instructive case: its container securityContext is
  `drop: ["ALL"], add: ["NET_BIND_SERVICE"]` with `seccompProfile: RuntimeDefault` and
  `allowPrivilegeEscalation: false` — three of the four Restricted rules, already satisfied. The
  **only** one it fails is `runAsNonRoot: true`, because the stock `corentinth/it-tools` image
  (`FROM nginx:stable-alpine`, no `USER` directive) needs to start as root to bind port 80 — no
  securityContext trick fixes that without a custom `nginx.conf`. So `it-tools`'s namespace runs
  at **Baseline** instead (see `apps/it-tools/components/namespace/namespace.yaml`) — not because
  the container is loosely secured, but because one specific, non-negotiable rule collides with
  how that particular public image happens to be built. See
  [`apps/it-tools/README.md`](../apps/it-tools/README.md#why-this-app-doesnt-use-the-restricted-pod-security-standard)
  for the full reasoning.

The practical takeaway: when you deploy your own app, check what its image actually needs (does
it run as root? does it bind a port below 1024?) before picking Restricted by default — and if it
can't meet all four rules, Baseline plus narrowing capabilities down to exactly what's needed
(same pattern as `it-tools`) is a reasonable middle ground, not a security failure.

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
