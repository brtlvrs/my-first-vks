# 04 — VCF-native services

[Chapter 03](03-required-platform-services.md) describes capabilities you might self-host
(Gitea, MinIO, a registry). Before standing any of those up, it's worth checking whether Broadcom
already provides an equivalent directly on your VCF Supervisor — that's what this chapter covers.

<!-- toc -->

- [what "Supervisor Services" means](#what-supervisor-services-means)
- [a concrete example: Data Protection (Velero)](#a-concrete-example-data-protection-velero)
- [checking what's available on your Supervisor](#checking-whats-available-on-your-supervisor)
- [bring-your-own vs. VCF-native: how to decide](#bring-your-own-vs-vcf-native-how-to-decide)

<!-- tocstop -->

## what "Supervisor Services" means

VCF 9.x Supervisors can offer a catalog of **Supervisor Services** — platform add-ons a vSphere
admin can enable centrally (registry, ingress controllers, cert-manager, and VMware's own Data
Protection service, among others), which then become available to every vSphere Namespace on
that Supervisor without each team standing up and operating their own copy. Think of it as an
app-marketplace layer sitting between "raw Supervisor" and "whatever you self-host."

Which services are actually offered, and whether your namespace has access to them, depends on
your VCF edition/licensing and on how the platform team configured the Supervisor — this is not
something a generic repo like this one can assume for you.

## a concrete example: Data Protection (Velero)

VMware's **Data Protection** Supervisor Service is a good example of what "VCF-native" actually
looks like in practice: it's consumed via a `VeleroService` custom resource, with the operator
itself running platform-wide, managed by Broadcom, not something you install yourself. This repo
doesn't provision it by default (see [chapter 11](11-backups-with-velero.md) for why), but if you
add it, every `CHANGE_ME_*` value on that CR is about *where it backs up to* (an S3-compatible
bucket — [chapter 03](03-required-platform-services.md)), not about standing up Velero itself —
that part is already handled for you, platform-wide, the moment the Supervisor Service is enabled.

## checking what's available on your Supervisor

Ask your platform/vSphere admin, or check the Supervisor's own UI for a services/catalog section.
From the CLI, once you have a context (see [chapter 06](06-connecting-to-supervisor.md)):

```
kubectl api-resources | grep -i vmware
```

Custom resources under a `*.vmware.com` API group are a strong signal of a Supervisor Service
being enabled and usable from your namespace — `veleroappoperator.vmware.com` (Data Protection,
see above) is one example; a registry or ingress operator would show up the same way if enabled.

## bring-your-own vs. VCF-native: how to decide

| | VCF-native (Supervisor Service) | Bring-your-own (chapter 03) |
|---|---|---|
| Who operates it | Broadcom / your platform team | You |
| Availability | Depends on VCF edition + what's enabled for your namespace | Always available — you control it |
| Best fit | Production, once you know it's offered and enabled | Learning this repo for the first time; environments without the service enabled |

For a team new to VKS, a reasonable default: use the `apps/`/`platform/` worked examples with
self-hosted services from [chapter 03](03-required-platform-services.md) while learning the
pattern, then switch specific `CHANGE_ME_*` values (bucket/endpoint, registry URL) to point at
the VCF-native equivalent once you've confirmed it's enabled for your namespace — the manifests
themselves don't change, only the values.
