# 14 — infrastructure prerequisites

**This chapter is for a different audience than 00-13.** Everything else in this cookbook
assumes a vSphere Namespace already exists and you have at least viewer access to it. This
chapter is what has to be true *before* that's the case — a checklist for whoever provisions the
Supervisor and its namespaces (a VI admin, NSX admin, or platform team). If that's not you, use
this as the list of things to request from whoever it is, not something to do yourself.

<!-- toc -->

- [networking: three ways to get there](#networking-three-ways-to-get-there)
- [vmClasses](#vmclasses)
- [content libraries](#content-libraries)
- [RBAC: getting your team access to a namespace](#rbac-getting-your-team-access-to-a-namespace)
- [checklist to hand to your VI/platform admin](#checklist-to-hand-to-your-viplatform-admin)
- [further reading](#further-reading)

<!-- tocstop -->

## networking: three ways to get there

A vSphere Namespace needs network connectivity — and specifically a load balancer, for the
Supervisor API and for any `type: LoadBalancer` Service your apps create — underneath it before it
can exist. As of VCF 9.1 there are three architecturally different ways to provide that. Which one
your Supervisor uses isn't something this repo can decide for you (it's a VI design choice, made
once for the whole Supervisor, not per-namespace), but it's worth knowing all three exist:

- **NSX overlay-backed (the traditional model).** A full NSX topology — Edge cluster, Tier-0/
  Tier-1 gateways, Geneve overlay tunnels between hosts. More capable and flexible (dynamic
  routing, more topology options), and more to operate and understand.
- **NSX Project + VPC, VLAN-backed via a Distributed Transit Gateway (DTGW) + Virtual Network
  Appliance (VNA) cluster — new in VCF 9.1.** This eliminates the need for a dedicated NSX Edge
  cluster entirely; per Broadcom's own description, it needs "only a VLAN and a routed network."
  The VNA cluster (a small HA appliance, minimum 2 nodes) is what makes this possible — a purely
  distributed architecture can't provide *stateful* services (NAT, layer-7 load balancing) on its
  own, so the VNA cluster exists specifically to host those, working alongside the DTGW. Still
  NSX underneath, just less of it to stand up than the overlay-backed model.
- **Foundation Load Balancer (FLB) — no NSX at all.** A native, lightweight Layer-4 load balancer
  that ships built into VVF/VCF at no extra license cost, running on plain **vSphere Distributed
  Switch (VDS)** networking. vCenter's own Lifecycle Manager deploys and manages it automatically
  as part of Supervisor activation — as a single VM for a resource-constrained/homelab-style
  environment, or an HA pair for production. **This is the practical option if you're running VCF
  9.1 on a standalone vCenter in a homelab and don't want to stand up NSX at all** — it needs
  nothing beyond a VDS and a routable IP range for the load balancer itself.

Nothing in `apps/` or `platform/` cares which model backs your namespace — the manifests are
identical either way. This is purely a Supervisor-level setup decision, made before any namespace
exists.

**If your Supervisor uses the VLAN-backed NSX path**, the prerequisites your VI/NSX admin needs in
place are, at a minimum:

- A physical VLAN with routing capability, for the external connection.
- A routable **External IP Block** (an IP range explicitly marked "External" visibility in NSX).
- Optionally, a **Private Transit Gateway IP Block**, if you'll have more than one VPC that needs
  to route to each other.
- The **VNA cluster** itself, deployed with at least 2 nodes for HA, selected when the DTGW is
  created.

**If your Supervisor uses FLB instead**, the prerequisites are simpler — a VDS-networked
Supervisor (no NSX Segment/VPC required) and a routable IP range for the load balancer to hand out
VIPs from. It's enabled during Supervisor activation (or added afterward to an existing
"Simplified Supervisor"), not something applied per-namespace.

This is evolving, VCF-9.1-specific material — see [further reading](#further-reading) below
rather than treating this chapter as the authority on NSX/load-balancer design.

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

- [ ] Decide networking for the Supervisor, once, not per-namespace: NSX overlay-backed, NSX VLAN-backed (DTGW + VNA), or Foundation Load Balancer (no NSX — the homelab/standalone-vCenter option)
- [ ] If NSX VLAN-backed: physical VLAN + routing, External IP Block, VNA cluster (2+ nodes)
- [ ] If FLB: VDS networking + a routable IP range for VIPs — no NSX Segment/VPC needed
- [ ] At least one `VirtualMachineClass` available to the target namespace (`kubectl get virtualmachineclass`)
- [ ] A Content Library synced with at least one usable VM image (`kubectl get virtualmachineimage`)
- [ ] A vSphere Namespace created, with a resource quota and storage policy assigned
- [ ] A `vsphere.local` group or AD group granted **Can Edit** (or **Owner**, `vsphere.local` only) on that namespace

## further reading

This is new (VCF 9.1) and evolving material — a mix of third-party technical deep-dives and
official Broadcom sources, not something verified end-to-end against a live environment by this
repo:

- [VMware Cloud Foundation Blog — Simplify Workload Connectivity and Enhance Network Scale and Performance with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/simplify-workload-connectivity-and-enhance-network-scale-and-performance-with-vcf-9-1/) (official)
- [sdn-warrior.org — VCF 9.1: VNA and VPCs](https://sdn-warrior.org/posts/vcf9.1-vna-vpc/)
- [Puneet Sharma — VCF 9.1 NSX Technical Deep Dive: DTGW and VNA Clusters](https://puneetsharma.blog/2026/05/18/vcf-9-1-nsx-technical-deep-dive-implementing-distributed-transit-gateways-dtgw-and-vna-clusters/)
- [VMware Cloud Foundation Blog — Choosing the Right Load Balancer for vSphere Supervisor in vSphere 9.0+ and VCF 9.0+](https://blogs.vmware.com/cloud-foundation/2026/07/13/choosing-the-right-load-balancer-for-vsphere-supervisor-in-vsphere-9-0-and-vmware-cloud-foundation-9-0/) (official — compares FLB, NSX-LB, and Avi)
- [Broadcom Knowledge Base — vSphere Foundation Load Balancer (FLB): Architecture, Network Requirements, and FAQ](https://knowledge.broadcom.com/external/article/438769/vsphere-foundation-load-balancer-flb-arc.html) (official)
- [Broadcom TechDocs — Deploying vSphere Supervisor with Foundation Load Balancer](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/vsphere-supervisor-installation-and-configuration/deploying-vsphere-supervisor-with-foundation-load-balancer.html) (official)
