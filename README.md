# my-first-vks

A customer-agnostic starting point for deploying apps and managing cluster lifecycle on a
VMware VCF 9.1 Supervisor (VKS). This repo doesn't belong to any one customer — every value
that would normally be site-specific (endpoints, IPs, hostnames, storage policies, credentials)
is a `CHANGE_ME_*` placeholder. Fork it, fill in the placeholders for your environment, and
you have a working repo on day one.

It exists to flatten the learning curve for a team that is new to `kubectl` and to VKS/Supervisor
concepts: the [`docs/`](docs/README.md) cookbook walks through everything from preparing your
workstation to deploying your first app, and the two content folders below are small, worked
examples you can literally copy and rename rather than build from scratch.

<!-- toc -->

- [layout](#layout)
- [quick start](#quick-start)
- [why two different folder shapes](#why-two-different-folder-shapes)

<!-- tocstop -->

## layout

```
platform/   # cluster lifecycle: provisioning a VKS cluster, enabling Velero backups per namespace
apps/       # deploying an application into a namespace / VKS cluster
docs/       # the cookbook — start at docs/README.md
mise.toml   # root env vars + pinned CLI versions
mise-tasks/ # shared mise tasks (context switching, docs, environment doctor)
```

## quick start

1. Read [`docs/00-introduction.md`](docs/00-introduction.md), then work through the cookbook in order —
   it's written to be read top to bottom the first time.
2. `mise run doctor` — checks that the CLIs this repo needs are installed.
3. Copy `platform/example-namespace/` to provision your first cluster, and `apps/hello-vks/` to
   deploy your first app. Both are fully worked examples with `CHANGE_ME_*` placeholders called
   out in their own READMEs.

## why two different folder shapes

`apps/` and `platform/` both use kustomize `base`/`overlay` separation, but they are laid out
differently on purpose:

- **`apps/`** is *component-first*: `<app>/base/` + `<app>/overlays/<namespace>/`. An app
  belongs to exactly one namespace at a time, so grouping by app first makes sense.
- **`platform/`** is *namespace-first*: `<namespace>/<service>/`, overlaying a centralized
  `platform/bases/<type>/`. A vSphere Supervisor namespace is the real tenancy boundary and
  typically hosts more than one platform-level service (a cluster, a Velero backup config, ...),
  so grouping by namespace first matches both how Supervisor itself is organized and how you'll
  actually navigate this repo during day-2 operations.

See [`docs/07-repo-structure-and-conventions.md`](docs/07-repo-structure-and-conventions.md) for the full explanation.
