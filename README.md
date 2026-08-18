# my-first-vks

**Current version: [`v0.2.1`](CHANGELOG.md#021---2026-08-17)** — see [`CHANGELOG.md`](CHANGELOG.md)
for release history, and [versioning](#versioning) below for what pulling a specific tag buys you.

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
- [versioning](#versioning)
- [license](#license)
- [disclaimer](#disclaimer)

<!-- tocstop -->

## layout

```
platform/   # cluster lifecycle: provisioning a VKS cluster per namespace
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
  `platform/bases/<type>/`. A vSphere Supervisor namespace is the real tenancy boundary and can
  host more than one platform-level service (a cluster, and — see
  [`docs/11-backups-with-velero.md`](docs/11-backups-with-velero.md) — optionally Velero backups),
  so grouping by namespace first matches both how Supervisor itself is organized and how you'll
  actually navigate this repo during day-2 operations.

See [`docs/07-repo-structure-and-conventions.md`](docs/07-repo-structure-and-conventions.md) for the full explanation.

## versioning

This repo is tagged with [Semantic Versioning](https://semver.org/) (`vMAJOR.MINOR.PATCH`) —
pull a specific tag rather than tracking `main`, so a later update to your fork is a deliberate
choice, not something that lands underneath you. See [`CHANGELOG.md`](CHANGELOG.md) for what
changed at each tag, and [`docs/13-next-steps.md#versioning-this-repo`](docs/13-next-steps.md#versioning-this-repo)
for what a major/minor/patch bump means here.

## license

[MIT](LICENSE) — permissive, no warranty. See [disclaimer](#disclaimer) below for what that means
in practice for this specific repo.

## disclaimer

**Use this repo at your own risk.** It drives real vSphere/VCF infrastructure — provisioning and
deleting VKS clusters, powering VMs on and off, resizing data disks, rotating credentials — and
several mise tasks are destructive by design (`cluster:delete`, `vm:delete`, and anything under
`*:apply`/`*:redeploy` that reconciles live state to match git). Nothing here has been validated
against *your* environment. Before running anything against infrastructure that matters:

- Read the mise task's `description` and the relevant [`docs/`](docs/README.md) chapter before
  running it — a task name alone doesn't convey its blast radius.
- Try it against a disposable namespace/cluster first, not production.
- Review every `CHANGE_ME_*` placeholder — a stale or wrong value (namespace, storage class, CA
  certificate) can silently target the wrong environment instead of failing loudly.

The [MIT license](LICENSE) already makes this formal: the software is provided "AS IS", without
warranty of any kind, and the author/contributors are not liable for any damages arising from its
use.
