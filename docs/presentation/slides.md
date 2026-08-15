---
marp: true
theme: default
paginate: true
size: 16:9
---
# my-first-vks

A customer-agnostic starting point for VKS on a VCF 9.1 Supervisor

Built for teams new to `kubectl` and VKS — a stepping stone, not a finished product

---

## The problem this solves

- VKS/Supervisor has real depth: namespaces, clusters, VM Service, vSphere Pods, RBAC, networking
- Most teams' first exposure is a blank terminal and a wall of `kubectl` flags
- This repo trades "figure it out from docs" for "copy a working example and fill in the blanks"

---

## Core concepts (chapter 00)

- **Supervisor** — the management layer you authenticate against first
- **vSphere Namespace** — the real tenancy boundary; most of this repo is organized around it
- **VKS cluster** — a full, independent Kubernetes cluster, provisioned inside a namespace
- **VM Service** — declarative standalone VMs (`VirtualMachine` CRs), no guest cluster needed
- **vSphere Pods** — containers running natively on the Supervisor, no guest cluster needed

---

## Repo layout

```
platform/   cluster lifecycle — provisioning VKS clusters per namespace
apps/       deploying things — containers, VMs, vSphere Pods
docs/       this cookbook — start at docs/README.md
mise.toml   root env vars + pinned CLI versions
mise-tasks/ shared mise tasks
```

Every site-specific value is a `CHANGE_ME_*` placeholder — no customer data lives here

---

## Two different folder shapes, on purpose

- **`apps/`** is *component-first*: `<app>/base/` + `<app>/overlays/<namespace>/`
  → an app belongs to one namespace at a time
- **`platform/`** is *namespace-first*: `bases/<type>/` + `<namespace>/<service>/`
  → a namespace is the real tenancy boundary and hosts multiple services

Same kustomize `base`/`overlay` idea, different question each answers first

---

## mise: one entry point for everything

- `[tools]` pins CLI versions (`kubectl`, `kubectx`, `k9s`, `jq`, `yq`, ...)
- `[env]` cascades down the folder tree — `cd` into an overlay, the right context resolves itself
- `mise run <task>` replaces memorizing long `kubectl`/`vcf` invocations
- `mise run doctor` — one command to check your workstation is ready

---

## The cookbook: 15 chapters

00 introduction · 01 workstation · 02 SSH keys · 03 platform services · 04 VCF-native services
05 secrets · 06 connect to Supervisor · 07 repo conventions · 08 provision a cluster
09 deploy your first app · 10 day-2 ops · 11 Velero backups (optional addon) · 12 troubleshooting
13 next steps · 14 infrastructure prerequisites (for your VI/NSX admin)

Read 00→12 in order the first time; 13/14 are reference, not sequential

---

## Worked examples you can copy

| Example | Demonstrates |
|---|---|
| `platform/example-namespace/` | provisioning a VKS cluster |
| `apps/hello-vks/` | a container, restricted-PSS-compliant |
| `apps/hello-vsphere-pod/` | the same container, as a native vSphere Pod |
| `apps/hello-vm/` | VM Service, with a persistent data disk across redeploys |
| `apps/it-tools/` | a real app, both ways at once — VKS *and* vSphere Pod |

---

## Quick start

```bash
mise run doctor                     # check your workstation
```

1. Read `docs/00-introduction.md`, then work through the cookbook in order
2. Copy `platform/example-namespace/` → provision your first cluster
3. Copy `apps/hello-vks/` (or `it-tools/`) → deploy your first app

Each copied folder's own README lists exactly which `CHANGE_ME_*` values to fill in

---

## What's next

- **Chapter 13** — GitOps, platform-as-a-product (versioned, tag-pinned bases), 12-factor apps
- **Chapter 14** — for your VI/NSX admin: networking options, vmClasses, Content Libraries, RBAC
- **`TODO.md`** — open items, tracked in the open rather than hidden

---

# Questions?

`docs/README.md` is the front door — start there.

