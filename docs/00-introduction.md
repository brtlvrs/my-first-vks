# 00 — introduction

## what is VKS / Supervisor

**VCF (VMware Cloud Foundation) 9.1** is Broadcom's private-cloud platform. One of its
components, the **Supervisor**, turns a vSphere cluster into a Kubernetes control plane in its
own right — it exposes a Kubernetes-compatible API you can talk to directly with `kubectl`, and
it can provision full, independent Kubernetes clusters on demand. That capability is called
**VKS (VMware Kubernetes Service)**.

A few concepts you'll see everywhere in this repo:

- **Supervisor** — the management layer. You authenticate against it (via the `vcf` CLI) before
  you can do anything else.
- **vSphere Namespace** — a tenancy boundary carved out of the Supervisor by a vSphere admin.
  Resource quotas, storage policies, and VM classes are assigned per namespace. This is the unit
  most of this repo is organized around.
- **VKS cluster** — a full, independent Kubernetes cluster, provisioned *inside* a vSphere
  Namespace. It has its own API endpoint and its own `kubectl` context, separate from the
  Supervisor's.
- **Workload / app** — something you deploy either directly into a Supervisor namespace, or into
  a VKS cluster that lives inside one.

A VKS cluster isn't the only thing the Supervisor can provision declaratively — two more
capabilities worth knowing exist, both consumed directly against the Supervisor namespace (no
guest cluster involved), and both demonstrated with a worked example under `apps/`:

- **VM Service** — provisions a standalone virtual machine the same way you'd provision anything
  else in Kubernetes: `kubectl apply` a `VirtualMachine` custom resource, rather than clicking
  through vCenter. Useful for workloads that aren't (yet) containerized. See `apps/hello-vm/`.
- **vSphere Pods** — ordinary-looking Pods that run natively on the Supervisor itself (each one
  its own lightweight VM-isolated sandbox), with no VKS guest cluster underneath them at all. See
  `apps/hello-vsphere-pod/`.

Concrete guidance on *when* to reach for one of these over a regular VKS-cluster app is still
being worked out — see [`TODO.md`](../TODO.md).

If none of that means much yet, that's expected — [chapter 06](06-connecting-to-supervisor.md)
and [chapter 07](07-repo-structure-and-conventions.md) build it up from first principles.

## what this repo gives you

This repo is not tied to any customer or environment — every site-specific value (endpoints,
IPs, storage policy names, credentials) is a `CHANGE_ME_*` placeholder. It gives you:

- **`platform/`** — a working, worked example of provisioning a VKS cluster for a namespace, using
  [kustomize](https://kustomize.io/) `base`/`overlay` manifests.
- **`apps/`** — a working, worked example of deploying an application, same pattern.
- **[`mise`](https://mise.jdx.dev/)** as the single entry point for both pinning CLI versions and
  running the day-to-day commands (`mise run <task>`), so you don't have to memorize long
  `kubectl`/`vcf` invocations.
- **This cookbook** — everything you need to go from an empty workstation to a deployed app,
  without prior Kubernetes or VKS experience.

## how the chapters fit together

```
01 prepare workstation ──▶ 02 SSH keys ──▶ 06 connect to Supervisor ──▶ 07 repo conventions
                                                                              │
                        ┌─────────────────────────────────────────────────────┤
                        ▼                                                     ▼
              08 provision a cluster                              09 deploy your first app
                        │                                                     │
                        └──────────────────────┬──────────────────────────────┘
                                                ▼
                                    10 day-2 operations
```

Chapter 11 (backups with Velero) sits outside this chain — it's an optional addon this repo
doesn't provision by default, not a required next step. See its own chapter for why.

Chapters 03, 04, and 05 (required platform services, VCF-native services, secret management) are
background you'll want *before* chapter 11, but they don't block chapters 08/09 — the worked
examples in `platform/` and `apps/` will run without them. Chapter 12 is a reference you'll come
back to whenever something doesn't behave as expected.

[Chapter 14](14-infrastructure-prerequisites.md) sits outside this flow entirely — it's written
for whoever provisions the vSphere Namespace this whole cookbook assumes already exists (a VI/NSX
admin), not for the team using chapters 00-13. Read it if that's you, or hand it to whoever it is.

[Chapter 15](15-further-reading.md) is a plain reference list — external links for the topics
this cookbook touches on, not something to read start to finish.
