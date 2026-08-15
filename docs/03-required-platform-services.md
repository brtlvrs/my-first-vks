# 03 — required platform services

This repo assumes a small number of *capabilities* exist somewhere in your environment. It
deliberately doesn't hard-wire you to a specific product for any of them — any tool that provides
the capability works. This chapter describes each capability, then walks through one concrete
example product per capability (Gitea, MinIO) so "any S3-compatible bucket" isn't just an
abstract phrase the first time you touch it.

Before standing any of these up yourself, check [chapter 04](04-vcf-native-services.md) — VCF's
Supervisor Services catalog may already provide some of them.

<!-- toc -->

- [capability 1: a git remote](#capability-1-a-git-remote)
- [capability 2: S3-compatible object storage](#capability-2-s3-compatible-object-storage)
- [capability 3: a container registry (optional)](#capability-3-a-container-registry-optional)
- [worked example: Gitea](#worked-example-gitea)
- [worked example: MinIO](#worked-example-minio)

<!-- tocstop -->

## capability 1: a git remote

**What it's for:** somewhere to push this repo (and its fork, once customized) so more than one
person can work from it, and so its history is durable.

**What this repo needs from it:** SSH (or HTTPS) clone/push access. Nothing kustomize- or
mise-specific — this is a plain git repo.

**Options:** GitHub, GitLab, a company-run Gitea/Forgejo instance, Bitbucket — anything that
speaks git. See [chapter 02](02-ssh-keys.md) for authenticating to it.

## capability 2: S3-compatible object storage

**What it's for:** Velero (the backup mechanism used in [`platform/bases/velero/`](../platform/bases/velero/))
needs an S3-compatible bucket to store backup data.

**What this repo needs from it:** a bucket, a region, an S3 API endpoint URL, and an access
key/secret key pair. These map directly onto the `CHANGE_ME_*` values in
[`platform/bases/velero/veleroservice.yaml`](../platform/bases/velero/veleroservice.yaml) and
each namespace overlay's `kustomization.yml` patch (`spec.bucket`, `spec.backuplocationconfig`),
plus a `cloud-credentials` Secret — see [chapter 11](11-backups-with-velero.md) for the full
walkthrough. It does **not** have to be real AWS S3 — the Velero AWS plugin works against any
S3-compatible API, which is exactly what makes a self-hosted option like MinIO viable.

## capability 3: a container registry (optional)

**What it's for:** hosting your own container images, if `apps/<your-app>/` ends up building a
custom image rather than using a public one (the `hello-vks` example pulls a public image and
needs nothing here).

**Options:** Docker Hub, GitHub Container Registry, a self-hosted Harbor instance, or — check
[chapter 04](04-vcf-native-services.md) first — a registry the Supervisor already provides.

## worked example: Gitea

[Gitea](https://about.gitea.io/) is a lightweight, self-hostable git server — a reasonable
default if you don't already have a company-standard one. Minimal self-hosted setup:

```
docker run -d --name gitea -p 3000:3000 -p 2222:22 \
  -v gitea-data:/data \
  gitea/gitea:latest
```

Then, from `http://localhost:3000` (or wherever you exposed it): create an admin account, create
an organization/repo, and add your SSH public key under **Settings → SSH/GPG Keys** (this is the
"git server's SSH-keys settings page" referenced at the end of [chapter 02](02-ssh-keys.md)).
Clone with:

```
git clone ssh://git@<your-gitea-host>:2222/<org>/<repo>.git
```

## worked example: MinIO

[MinIO](https://min.io/) is a self-hostable, S3-API-compatible object store — a reasonable
default for the Velero bucket if you don't already have on-prem S3-compatible storage. Minimal
self-hosted setup:

```
docker run -d --name minio -p 9000:9000 -p 9001:9001 \
  -e "MINIO_ROOT_USER=CHANGE_ME_ACCESS_KEY" \
  -e "MINIO_ROOT_PASSWORD=CHANGE_ME_SECRET_KEY" \
  -v minio-data:/data \
  minio/minio server /data --console-address ":9001"
```

Create the bucket Velero will use (via the console at `http://localhost:9001`, or the `mc` CLI):

```
mc alias set local http://localhost:9000 CHANGE_ME_ACCESS_KEY CHANGE_ME_SECRET_KEY
mc mb local/CHANGE_ME_BUCKET
```

The resulting values map directly onto the placeholders in
[`platform/example-namespace/velero/kustomization.yml`](../platform/example-namespace/velero/kustomization.yml):
`spec.bucket` = your bucket name, `s3Url` = your MinIO endpoint, `region` = any string MinIO
accepts (it doesn't enforce real AWS regions) — see [chapter 11](11-backups-with-velero.md) for
turning the access/secret key pair into the `cloud-credentials` Secret Velero expects.
