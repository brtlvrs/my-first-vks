# 11 — backups with Velero (optional addon)

**Not included in this repo by default.** Backups are a real, worthwhile addition for a
production setup — but they're an *addon* on top of the core pattern this repo teaches, not
something every team needs from day one, so `platform/` doesn't ship a working example the way it
does for cluster provisioning. This chapter describes the shape it would take, as something worth
looking into once the basics from chapters 08/09 feel routine.

<!-- toc -->

- [what it would involve](#what-it-would-involve)
- [where it would fit in this repo](#where-it-would-fit-in-this-repo)
- [what you'd need first](#what-youd-need-first)

<!-- tocstop -->

## what it would involve

Backups for a vSphere Namespace go through the Supervisor's **Data Protection** service — see
[chapter 04](04-vcf-native-services.md) — exposed as a `VeleroService` custom resource
(`veleroappoperator.vmware.com/v1alpha1`). At a high level, the CR looks like:

```yaml
apiVersion: veleroappoperator.vmware.com/v1alpha1
kind: VeleroService
metadata:
  name: <namespace>-velero
  namespace: <your-namespace>
spec:
  objectstoreprovider: aws   # works against any S3-compatible backend, not just real AWS
  bucket: <your-bucket>
  backuplocationconfig: region=<region>,s3ForcePathStyle=true,s3Url=https://<your-s3-endpoint>
  nosecret: true             # expects a pre-existing "cloud-credentials" Secret — see chapter 05
  usevolumesnapshots: true
```

The operator itself runs platform-wide, managed by Broadcom — applying this CR is the whole
"install," nothing else to deploy. The pattern for the Secret it expects
(`cloud-credentials`, containing a standard AWS-style credentials file) is exactly the baseline
pattern from [chapter 05](05-secret-management.md): create it imperatively, never commit it.

## where it would fit in this repo

`platform/`'s namespace-first layout is built to hold more than one service per namespace — see
[chapter 07](07-repo-structure-and-conventions.md) — specifically so something like this slots in
the same way `vks-cluster/` already does: a `platform/bases/velero/` base (the CR above, with
`CHANGE_ME_*` placeholders for bucket/region/endpoint/name), and a
`platform/<namespace>/velero/` overlay per namespace that wants it. No new pattern to learn, just
another service folder alongside the cluster.

## what you'd need first

- An S3-compatible bucket, region, and endpoint — see
  [chapter 03](03-required-platform-services.md#capability-2-s3-compatible-object-storage-only-if-you-add-the-velero-addon)
  (the MinIO worked example there produces exactly these values), or check
  [chapter 04](04-vcf-native-services.md) for whether your Supervisor already offers
  S3-compatible storage natively.
- Confirmation that Data Protection is actually enabled on your Supervisor and available to your
  namespace (`kubectl api-resources | grep veleroappoperator` — see
  [chapter 04](04-vcf-native-services.md#checking-whats-available-on-your-supervisor)).
- The exact plugin image references (`spec.plugins`, omitted from the sketch above) depend on your
  Supervisor's specific Data Protection operator version — worth pulling from
  `kubectl get cm velero-vsphere-plugin-config -n svc-velero-<id> -o yaml` on the target
  Supervisor rather than assuming a fixed version.

If you build this out, see [`TODO.md`](../TODO.md) — it's tracked there as an open item, and this
chapter is a reasonable place to turn back into a full worked example once someone has.
