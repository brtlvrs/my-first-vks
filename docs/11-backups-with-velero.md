# 11 — backups with Velero

Ties together [chapter 03](03-required-platform-services.md) (where the S3 bucket comes from) and
[chapter 05](05-secret-management.md) (how its credentials get into the cluster). Backups for a
vSphere Namespace go through the Supervisor's Data Protection service — see
[chapter 04](04-vcf-native-services.md) — exposed as a `VeleroService` custom resource, following
the same `bases/` + `<namespace>/velero/` pattern used for clusters themselves.

<!-- toc -->

- [1. have a bucket ready](#1-have-a-bucket-ready)
- [2. create the credentials secret](#2-create-the-credentials-secret)
- [3. fill in the placeholders](#3-fill-in-the-placeholders)
- [4. apply](#4-apply)
- [5. verify](#5-verify)

<!-- tocstop -->

## 1. have a bucket ready

You need an S3-compatible bucket, region, and endpoint URL — see
[chapter 03](03-required-platform-services.md#capability-2-s3-compatible-object-storage) (the
MinIO worked example there produces exactly these values) or
[chapter 04](04-vcf-native-services.md) if your Supervisor offers Data Protection storage
natively.

## 2. create the credentials secret

The `VeleroService` CR is created with `nosecret: true` (see
[`platform/bases/velero/veleroservice.yaml`](../platform/bases/velero/veleroservice.yaml)), which
means it expects a Secret named `cloud-credentials` to already exist in the target namespace —
this is the baseline pattern from [chapter 05](05-secret-management.md):

```
kubectl create secret generic cloud-credentials -n <your-namespace> --from-file=cloud=<path-to-credentials-file>
```

where `<path-to-credentials-file>` is a standard AWS-style credentials file:

```
[default]
aws_access_key_id=...
aws_secret_access_key=...
```

## 3. fill in the placeholders

In `platform/<your-namespace>/velero/kustomization.yml`, update the JSON6902 patch values:
`/metadata/name`, `/spec/bucket`, and `/spec/backuplocationconfig` (`region=...,s3ForcePathStyle=true,s3Url=https://...`).
Also update `namespace:` at the top of the file to match your real namespace.

## 4. apply

```
cd platform/<your-namespace>/velero
mise run cluster:render   # sanity-check first
mise run cluster:apply
```

## 5. verify

```
kubectl get veleroservice -n <your-namespace>
kubectl describe veleroservice <name> -n <your-namespace>
```

A healthy `VeleroService` should reach a ready/reconciled status within a minute or two of the
`cloud-credentials` Secret and bucket both being valid. If it doesn't, see
[chapter 12](12-troubleshooting-cookbook.md).

**Known caveat carried over from this repo's source pattern:** the plugin image references in
`platform/bases/velero/veleroservice.yaml` were read off a live Supervisor's default operator
config, not confirmed end-to-end against a real install on every VCF/Velero-operator version — see
[`TODO.md`](../TODO.md). Re-verify them against your own Supervisor if backups don't come up
cleanly.
