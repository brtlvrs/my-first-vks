# 05 — secret management

Credentials — VM login passwords, registry pull secrets, app secrets — must never be committed to
git. This chapter covers the baseline pattern this repo already uses, and two upgrade paths for
when the baseline stops being enough.

<!-- toc -->

- [baseline: imperative secrets + gitignore](#baseline-imperative-secrets--gitignore)
- [upgrade path 1: sealed-secrets](#upgrade-path-1-sealed-secrets)
- [upgrade path 2: Vault](#upgrade-path-2-vault)
- [which one should you use](#which-one-should-you-use)

<!-- tocstop -->

## baseline: imperative secrets + gitignore

The pattern already in use in this repo (see
[`apps/hello-vm/base/vm.yaml`](../apps/hello-vm/base/vm.yaml)'s `hashed_passwd`): the manifest
declares that it *expects* a Secret to already exist, but the Secret itself is created
imperatively, outside of `kubectl apply -k`, and is never written to a YAML file that gets
committed:

```
kubectl create secret generic hello-vm-passwd -n <namespace> --from-literal=password="$(openssl passwd -6)"
```

(or `mise run vm:set-password` from the overlay folder — see
[`apps/hello-vm/README.md`](../apps/hello-vm/README.md)). The same shape applies to anything
else that needs a pre-existing Secret — a Velero `cloud-credentials` Secret if you add that addon
(see [chapter 11](11-backups-with-velero.md)), a registry pull secret, an app's own API key.

If you do need to keep a secret manifest around locally (for repeat use, or to hand to a
teammate), name it `secret.yaml` — this repo's [`.gitignore`](../.gitignore) already excludes
that filename everywhere, so it can't be accidentally committed.

This is simple, has zero extra infrastructure, and is entirely adequate for a small team learning
the pattern or running a handful of environments. Its limits: no audit trail of who created or
rotated what, no automatic rotation, and every secret has to be re-created by hand on a fresh
checkout (nothing in git describes it, by design).

## upgrade path 1: sealed-secrets

[Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) lets you encrypt a
Secret client-side into a `SealedSecret` custom resource that *is* safe to commit — only the
in-cluster controller (holding the private key) can decrypt it back into a real Secret. This gets
you GitOps-style "the whole desired state is in git" back, at the cost of running one extra
controller in the cluster and the `kubeseal` CLI locally. A good middle ground when the baseline's
"nothing in git" starts to hurt (you want history/diffs on *when* a secret changed, even if not
its value) but standing up Vault feels like overkill.

## upgrade path 2: Vault

[HashiCorp Vault](https://www.vaultproject.io/) (or a Vault-compatible service) centralizes
secret storage, access policies, rotation, and audit logging. Kubernetes consumes it via either
the [External Secrets Operator](https://external-secrets.io/) (syncs a Vault path into a regular
Secret on a schedule) or the [Vault CSI provider](https://developer.hashicorp.com/vault/docs/platform/k8s/csi)
(mounts secrets directly into pods, never materializing a Secret object at all). This is the right
tool once you have real rotation/audit/compliance requirements, or secrets shared across more
systems than just this repo's clusters — but it's genuinely more infrastructure to run and
operate, and not something to reach for on day one.

[Chapter 02](02-ssh-keys.md) covers generating and protecting your own SSH key locally; storing
that key centrally in Vault (generate-or-fetch on a new machine, rather than re-generating by
hand) is a natural extension once Vault is in place, but is out of scope for this repo's initial
worked examples — see [`TODO.md`](../TODO.md).

## which one should you use

| Situation | Use |
|---|---|
| Learning this repo, small team, few environments | Baseline (imperative + gitignore) |
| Want git history/diffs on secret changes, without new infra to operate | sealed-secrets |
| Real rotation/audit/compliance needs, secrets shared beyond this repo | Vault |

You can adopt these incrementally and mix them — e.g. Vault for org-wide credentials, baseline
for a one-off local test secret. Nothing in `apps/` or `platform/`'s manifests has to change to
move between them; only how the Secret object gets into the cluster changes.
