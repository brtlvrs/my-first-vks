# 07 — repo structure and conventions

<!-- toc -->

- [apps/ vs platform/](#apps-vs-platform)
- [how mise [env] cascades](#how-mise-env-cascades)
- [the CHANGE_ME convention](#the-change_me-convention)
- [an alternative pattern you'll see referenced: path-derived context](#an-alternative-pattern-youll-see-referenced-path-derived-context)

<!-- tocstop -->

## apps/ vs platform/

Both folders use kustomize `base`/`overlay` separation — a `base/` holds the generic manifest
with placeholder values, and an `overlay/` (or several) patches it for one specific target. But
they're shaped differently:

- **`apps/`** is *component-first*: `<app>/base/` + `<app>/overlays/<namespace>/`. Browsing by
  app name makes sense because an app belongs to exactly one namespace at a time — "what is
  `hello-vks` and where does it run" is one folder.
- **`platform/`** is *namespace-first*: `bases/<type>/` (shared, one per service kind) plus
  `<namespace>/<service>/` overlays. A vSphere Namespace is a real tenancy boundary that
  typically hosts *multiple* platform-level things at once (a cluster, its Velero config) — so
  "what's running in this namespace" is the more useful question, and the folder layout answers
  it directly. This also matches how the production repos this project is based on
  (`vks-platform`) are organized in practice.

Neither is "more correct" in general — they're each shaped around the question you'll actually
ask most often for that kind of resource.

## how mise [env] cascades

mise merges `[env]` blocks from every `mise.toml` between the repo root and your current
directory, with the closer one winning on conflicts. Concretely:

```
mise.toml                                  VCF_ENDPOINT, VCF_SUPERVISOR
platform/example-namespace/mise.toml       + VCF_NAMESPACE
platform/example-namespace/vks-cluster/    + VCF_CLUSTER
  mise.toml
```

Run `mise run mise:env` from inside `platform/example-namespace/vks-cluster/` and you'll see all
four variables resolved — you only ever set each one once, at the shallowest folder where it's
already correct for everything beneath it.

## the CHANGE_ME convention

Every value in this repo that's specific to a real customer/environment is written as
`CHANGE_ME_SOMETHING` (or, in a worked-example overlay that's already "filled in" to demonstrate
the pattern, an obviously-fake placeholder like `example-bucket`). Each folder's own `README.md`
carries a table listing every placeholder it introduces and what to replace it with — check
[`platform/README.md`](../platform/README.md) and [`apps/README.md`](../apps/README.md) before
copying an example folder for real use.

## an alternative pattern you'll see referenced: path-derived context

If you go on to look at a production repo this project is modeled on, you may see `VCF_NAMESPACE`
and `VCF_CLUSTER` derived *automatically* from folder depth, via a Tera template in the root
`mise.toml`, rather than set explicitly per folder like here. It's a neat trick, but it has a
sharp edge: anything else sitting at that same folder depth (a `velero/` folder alongside a
cluster folder, for instance) gets swept up by the same derivation and silently resolves to a
nonsense cluster name. This repo uses explicit `[env]` values in each folder's own `mise.toml`
instead — one extra line per folder, but no folder-depth "magic" to explain to someone new to the
pattern, and no risk of that specific class of mistake.
