# it-tools

A real public app — [it-tools](https://github.com/CorentinTh/it-tools), a self-hosted web
toolbox — deployed two ways from one shared `base/`: onto a VKS guest cluster
(`overlays/example-namespace-vks/`) and directly onto the Supervisor as a vSphere Pod
(`overlays/example-namespace-vpod/`). Compare against `../hello-vks/` and
`../hello-vsphere-pod/`, which are the same split but for a synthetic app — this one shows the
same pattern applied to something you didn't write yourself.

<!-- toc -->

- [why this app doesn't use the Restricted Pod Security Standard](#why-this-app-doesnt-use-the-restricted-pod-security-standard)
- [how the two variants share one base](#how-the-two-variants-share-one-base)
- [deploying](#deploying)
- [CHANGE_ME placeholders](#change_me-placeholders)

<!-- tocstop -->

## why this app doesn't use the Restricted Pod Security Standard

`hello-vks/` and `hello-vsphere-pod/` both enforce the **Restricted** Pod Security Standard and
run as a non-root user — that works because they use `nginxinc/nginx-unprivileged`, an image
specifically built to listen on port 8080 as a non-root user. `it-tools`'s actual published image
is different: it's built `FROM nginx:stable-alpine` with no `USER` directive (so it starts as
root) and a hardcoded `listen 80` inside its baked-in `nginx.conf`, with no environment variable
to change it. Binding a port below 1024 needs the `NET_BIND_SERVICE` capability — something only
root has by default, and something the Restricted profile's `runAsNonRoot: true` requirement rules
out entirely.

So this app's namespace enforces **Baseline** instead of Restricted (see
`components/namespace/namespace.yaml`), and `base/deployment.yaml`'s container keeps root, but
**drops every capability except the one it actually needs**:

```yaml
capabilities:
  drop: ["ALL"]
  add: ["NET_BIND_SERVICE"]
```

This is a real, common situation — not every public image is built rootless, and forcing
`runAsNonRoot: true` on one that isn't doesn't change what the image does, it just crash-loops on
a permission-denied port bind. Knowing when to reach for Baseline instead of Restricted, and how
to narrow root down to exactly the one capability an image needs, is the actual skill — not
"always use Restricted."

## how the two variants share one base

Both overlays reference the same `../../base/` (one `Deployment`, one `Service` — no `Namespace`
resource in `base/` itself, see `apps/hello-vm/README.md`'s comment on the same pattern) and
differ only in which [kustomize component](https://kubectl.docs.kubernetes.io/guides/config_management/components/)
they mix in:

- `overlays/example-namespace-vks/` uses the `components/namespace/` component — a VKS guest
  cluster lets you create your own app-specific namespace, so this adds one (`it-tools`, Baseline
  PSS).
- `overlays/example-namespace-vpod/` uses the `components/vsphere-pod/` component instead — a
  JSON6902 patch adding `runtimeClassName: runv` to the Deployment, the field that actually makes
  it a vSphere Pod. It deploys straight into the vSphere Namespace (no separate app namespace),
  same as `../hello-vsphere-pod/`.

Neither overlay uses the other's component — setting `runtimeClassName: runv` on a normal VKS
guest cluster would fail scheduling outright (no such `RuntimeClass` registered there), and the
`vpod` variant has nowhere to put a separate app namespace even if it wanted one.

## deploying

From whichever overlay matches what you're targeting:

```
cd overlays/example-namespace-vks    # or overlays/example-namespace-vpod
mise run app:render                  # sanity-check
mise run context:supervisor          # or context:use / context:cluster, if already logged in
mise run app:apply
```

## CHANGE_ME placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `example-namespace` (folder + `VCF_NAMESPACE`) | both overlays' `mise.toml` | your real vSphere Namespace |
| `example-cluster` (folder + `VCF_CLUSTER`) | `overlays/example-namespace-vks/mise.toml` | your real VKS cluster name |
| `service.beta.kubernetes.io/vsphere-load-balancer-class: "nsx"` | `base/service.yaml` | your Supervisor's actual load-balancer class, if different |
