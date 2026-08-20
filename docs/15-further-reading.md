# 15 — further reading

A curated list of external resources for the topics this cookbook touches on or assumes — not
required reading, and this repo doesn't depend on any of it. Where to go from here.

<!-- toc -->

- [VCF / vSphere / NSX / VKS official docs](#vcf--vsphere--nsx--vks-official-docs)
- [broader DevOps / Linux / Kubernetes fundamentals](#broader-devops--linux--kubernetes-fundamentals)
- [software design philosophy](#software-design-philosophy)
- [GitOps](#gitops)
- [tools this repo uses directly](#tools-this-repo-uses-directly)

<!-- tocstop -->

## VCF / vSphere / NSX / VKS official docs

- [Broadcom TechDocs — VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html)
  — the authoritative source for anything this cookbook simplifies or omits. Chapters 04 and 14
  in particular only scratch the surface of Supervisor Services and networking.
- [Broadcom TechDocs — vSphere Foundation Load Balancer (FLB)](https://knowledge.broadcom.com/external/article/438769/vsphere-foundation-load-balancer-flb-arc.html)
  and [Deploying vSphere Supervisor with FLB](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/vsphere-supervisor-installation-and-configuration/deploying-vsphere-supervisor-with-foundation-load-balancer.html)
  — referenced in [chapter 14](14-infrastructure-prerequisites.md).

## broader DevOps / Linux / Kubernetes fundamentals

This repo deliberately covers only the VKS-specific slice — see
[chapter 13](13-next-steps.md#building-broader-team-fundamentals). If some of your team is newer
to the wider field:

- [90DaysOfDevOps](https://github.com/LondheShubham153/90DaysOfDevOps) — a free, structured,
  day-by-day curriculum covering Linux, git, Docker, Kubernetes, cloud fundamentals, and
  scripting from the ground up.
- [kubernetes.io/docs](https://kubernetes.io/docs/home/) — the Kubernetes project's own docs;
  worth reading directly rather than only ever meeting Kubernetes concepts secondhand through a
  VKS-specific lens.

## software design philosophy

- [12factor.net](https://12factor.net/) — the 12-factor app methodology. Referenced in
  [chapter 13](13-next-steps.md#12-factor-apps) against `apps/hello-vks/` specifically; worth
  reading in full once, not just the factors this repo happens to touch on.
- [agilemanifesto.org](https://agilemanifesto.org/) — the Agile Manifesto and its twelve
  principles. Not something this repo implements or enforces, but the underlying "working
  software over comprehensive documentation," "responding to change over following a plan"
  mindset is the same spirit behind this repo's worked-examples-over-reference-manual approach.
- [semver.org](https://semver.org/) and [keepachangelog.com](https://keepachangelog.com/) — the
  versioning scheme and changelog format this repo itself uses; see
  [chapter 13](13-next-steps.md#versioning-this-repo).

## GitOps

- [opengitops.dev](https://opengitops.dev/) — the CNCF GitOps Working Group's vendor-neutral
  definition: four principles (declarative, versioned and immutable, pulled automatically,
  continuously reconciled), independent of any specific tool. Read this before either of the
  tools below, since it's the model both of them implement.
- [fluxcd.io](https://fluxcd.io/) and [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/)
  — the two mainstream GitOps controllers, either of which is what
  [chapter 13](13-next-steps.md#from-manual-apply-to-gitops) means by "a controller reconciling
  the cluster against git."

## tools this repo uses directly

- [mise.jdx.dev](https://mise.jdx.dev/) — CLI version pinning and task runner, the single entry
  point for everything in this repo (see [chapter 01](01-prepare-your-workstation.md)).
- [kubernetes.io/docs/reference/kubectl](https://kubernetes.io/docs/reference/kubectl/) — the
  Kubernetes CLI itself.
- [github.com/ahmetb/kubectx](https://github.com/ahmetb/kubectx) — `kubectx`/`kubens`, for
  switching contexts and namespaces. Both pick up `fzf` (below) automatically for an interactive
  fuzzy picker if it's on `PATH` — no flag or config needed (set `KUBECTX_IGNORE_FZF=1` to opt
  back out).
- [k9scli.io](https://k9scli.io/) — terminal UI for browsing a cluster.
- [jqlang.org](https://jqlang.org/) — the JSON processor several tasks pipe `kubectl -o json`
  through.
- [github.com/mikefarah/yq](https://github.com/mikefarah/yq) — the YAML equivalent of `jq`; this
  is the Go `mikefarah/yq` mise installs, not the Python `kislyuk/yq` wrapper of the same name.
- [kustomize.io](https://kustomize.io/) — the `base`/`overlay`/`component` pattern every manifest
  in `apps/` and `platform/` is built on (see [chapter 07](07-repo-structure-and-conventions.md)).
- [github.com/junegunn/fzf](https://github.com/junegunn/fzf) — the fuzzy picker behind
  `cluster:delete` (see [chapter 10](10-day2-operations.md#deleting-a-cluster)) and the `kubectx`/
  `kubens` integration above.
- [marp.app](https://marp.app/) — the slide-deck format behind
  [`docs/presentation/slides.md`](presentation/slides.md).
