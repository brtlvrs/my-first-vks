# 14 — infrastructure prerequisites

**This chapter is for a different audience than 00-13.** Everything else in this cookbook
assumes a vSphere Namespace already exists and you have at least viewer access to it. This
chapter is what has to be true *before* that's the case — a checklist for whoever provisions the
Supervisor and its namespaces (a VI admin, NSX admin, or platform team). If that's not you, use
this as the list of things to request from whoever it is, not something to do yourself.

<!-- toc -->

- [networking: two ways to get there](#networking-two-ways-to-get-there)
- [vmClasses](#vmclasses)
- [content libraries](#content-libraries)
- [RBAC: getting your team access to a namespace](#rbac-getting-your-team-access-to-a-namespace)
- [checklist to hand to your VI/platform admin](#checklist-to-hand-to-your-viplatform-admin)
- [further reading](#further-reading)

<!-- tocstop -->

## networking: two ways to get there

A vSphere Namespace (and the VPC it sits on) needs network connectivity underneath it before it
can exist. As of VCF 9.1 there are two architecturally different ways to provide that — which one
your Supervisor uses isn't something this repo can decide for you (it's an NSX/VI design choice,
made once for the whole Supervisor, not per-namespace), but it's worth knowing both exist:

- **NSX overlay-backed (the traditional model).** A full NSX topology — Edge cluster, Tier-0/
  Tier-1 gateways, Geneve overlay tunnels between hosts. More capable and flexible (dynamic
  routing, more topology options), and more to operate and understand.
- **NSX Project + VPC, VLAN-backed via a Distributed Transit Gateway (DTGW) + Virtual Network
  Appliance (VNA) cluster — new in VCF 9.1.** This eliminates the need for a dedicated NSX Edge
  cluster entirely; per Broadcom's own description, it needs "only a VLAN and a routed network."
  The VNA cluster (a small HA appliance, minimum 2 nodes) is what makes this possible — a purely
  distributed architecture can't provide *stateful* services (NAT, layer-7 load balancing) on its
  own, so the VNA cluster exists specifically to host those, working alongside the DTGW. The
  result is meaningfully less networking complexity to stand up and reason about than the
  overlay-backed model, at the cost of being newer and less battle-tested.

Nothing in `apps/` or `platform/` cares which model backs your namespace — the manifests are
identical either way. This is purely a Supervisor-level setup decision, made before any namespace
exists.

**If your Supervisor uses the VLAN-backed path**, the prerequisites your VI/NSX admin needs in
place are, at a minimum:

- A physical VLAN with routing capability, for the external connection.
- A routable **External IP Block** (an IP range explicitly marked "External" visibility in NSX).
- Optionally, a **Private Transit Gateway IP Block**, if you'll have more than one VPC that needs
  to route to each other.
- The **VNA cluster** itself, deployed with at least 2 nodes for HA, selected when the DTGW is
  created.

This is evolving, VCF-9.1-specific material — see [further reading](#further-reading) below
rather than treating this chapter as the authority on NSX design.

## vmClasses

A `VirtualMachineClass` is a CPU/memory sizing tier — the same concept as an AWS instance type,
scoped to this Supervisor. They're defined once (by the VI admin) and then made available to
specific namespaces. **Both** VKS cluster nodes (`platform/bases/vks-cluster/cluster.yaml`'s
`vmClass` variable) and standalone VM Service VMs (`apps/hello-vm/base/vm.yaml`'s `className`)
consume them — it's the same resource either way.

Once you have namespace access: `kubectl get virtualmachineclass` lists what's available to you.
If nothing shows up, or the sizes you need aren't there, that's a request back to your VI admin,
not something fixable from inside the namespace.

## content libraries

A **Content Library** is where VM images (OS templates used by VM Service, e.g. the Ubuntu cloud
image `apps/hello-vm/base/vm.yaml`'s `imageName` expects) come from. The VI admin subscribes the
Supervisor to a content library (often VMware's own published one, or an internal one for
custom-built images) and syncs it, which is what makes `kubectl get virtualmachineimage` return
anything at all inside a namespace.

If `CHANGE_ME_VM_IMAGE` in `apps/hello-vm/` has no real value to reach for, that's a sign either
no content library is synced yet, or the one that is doesn't have an image you want — both are
VI-admin-side fixes.

## RBAC: getting your team access to a namespace

Two ways to grant a group of people access to a vSphere Namespace, depending on whether vCenter's
SSO is already wired up to an external identity source:

- **A `vsphere.local` local group** — created directly in vCenter's own SSO domain. Reasonable
  when there's no AD/LDAP identity source federated into vCenter yet, or for a small
  platform-team-only namespace.
- **An existing Active Directory (or other federated) security group** — if vCenter SSO is
  already wired to your org's AD, prefer this over creating `vsphere.local` accounts by hand:
  access then follows your existing AD group membership/lifecycle (onboarding, offboarding,
  audits) instead of a second, parallel system to keep in sync.

Either way, the assignment happens the same way: **vSphere Client → Workload Management →
Namespaces → `<namespace>` → Permissions → Add**, select the group, and assign a role —
**Can View**, **Can Edit**, or **Owner**. Two things worth knowing before you pick a role:

- Assign the role to a **group**, not individual user accounts — so future access changes go
  through group membership rather than per-user permission edits in vCenter.
- As already noted in [chapter 06](06-connecting-to-supervisor.md#roles-and-permissions): the
  **Owner** role doesn't work correctly for AD accounts on VCF 9.1 — use **Can Edit** for an AD
  group instead, and reserve `vsphere.local` accounts (where Owner does work as expected) for
  cases that specifically need it.

## checklist to hand to your VI/platform admin

A minimal, copy-pasteable version of everything above:

- [ ] Decide NSX overlay-backed vs. VLAN-backed (DTGW + VNA) networking for the Supervisor (once, not per-namespace)
- [ ] If VLAN-backed: physical VLAN + routing, External IP Block, VNA cluster (2+ nodes)
- [ ] At least one `VirtualMachineClass` available to the target namespace (`kubectl get virtualmachineclass`)
- [ ] A Content Library synced with at least one usable VM image (`kubectl get virtualmachineimage`)
- [ ] A vSphere Namespace created, with a resource quota and storage policy assigned
- [ ] A `vsphere.local` group or AD group granted **Can Edit** (or **Owner**, `vsphere.local` only) on that namespace

## further reading

This is new (VCF 9.1) and evolving material — these are third-party technical deep-dives plus one
official source, not something verified end-to-end against a live environment by this repo:

- [VMware Cloud Foundation Blog — Simplify Workload Connectivity and Enhance Network Scale and Performance with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/simplify-workload-connectivity-and-enhance-network-scale-and-performance-with-vcf-9-1/) (official)
- [sdn-warrior.org — VCF 9.1: VNA and VPCs](https://sdn-warrior.org/posts/vcf9.1-vna-vpc/)
- [Puneet Sharma — VCF 9.1 NSX Technical Deep Dive: DTGW and VNA Clusters](https://puneetsharma.blog/2026/05/18/vcf-9-1-nsx-technical-deep-dive-implementing-distributed-transit-gateways-dtgw-and-vna-clusters/)
