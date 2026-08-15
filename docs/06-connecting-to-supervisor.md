# 06 — connecting to the Supervisor

<!-- toc -->

- [roles and permissions](#roles-and-permissions)
- [get the CA certificate](#get-the-ca-certificate)
- [set your endpoint and Supervisor name](#set-your-endpoint-and-supervisor-name)
- [first-time login to the Supervisor](#first-time-login-to-the-supervisor)
- [logging in to a VKS cluster](#logging-in-to-a-vks-cluster)
- [re-authenticating later](#re-authenticating-later)

<!-- tocstop -->

## roles and permissions

Authentication is against the Supervisor. You need at least the **namespace viewer** role on the
vSphere Namespace you're working in (ask your platform/vSphere admin to assign it — ideally to an
AD/SSO group rather than per-user, so access is managed centrally). Note: the **namespace owner**
role doesn't work for AD user accounts on VCF 9.1 — use Kubernetes `RoleBinding`s inside the VKS
cluster itself for any finer-grained RBAC you need beyond namespace-level access.

## get the CA certificate

The Supervisor's API uses a certificate the `vcf` CLI needs to trust explicitly. Download it from
the Supervisor's own web UI (`https://<VCF_ENDPOINT>/`) and save it as `vcsa-ca.pem` in the repo
root — `mise-tasks/context.toml`'s tasks expect it there (`${MISE_PROJECT_ROOT}/vcsa-ca.pem`).

**Never commit this file.** [`.gitignore`](../.gitignore) already excludes `*-ca.pem`, but double
check `git status` after adding it the first time.

## set your endpoint and Supervisor name

The root [`mise.toml`](../mise.toml) ships `CHANGE_ME_VCF_ENDPOINT`/`CHANGE_ME_VCF_SUPERVISOR`
placeholders. Rather than editing that committed file with your real values, create a
`.mise.local.toml` next to it (already gitignored):

```toml
[env]
VCF_ENDPOINT = "https://<your-supervisor-address>"
VCF_SUPERVISOR = "<your-supervisor-name>"
```

mise merges this over the committed `mise.toml` automatically — every task now sees your real
values, and the committed file stays generic for the next person/customer who forks this repo.
Confirm with `mise run mise:env`.

## first-time login to the Supervisor

From anywhere in the repo:

```
mise run context:supervisor
```

This deletes any stale context of the same name first, then creates a fresh one and prompts for
your SSO username/password. Equivalent raw command, if you ever need it without mise:

```
vcf context create <VCF_SUPERVISOR> --endpoint <VCF_ENDPOINT> --auth-type basic --ca-certificate ./vcsa-ca.pem -u <username>
```

## logging in to a VKS cluster

Run from inside the specific `platform/<namespace>/<cluster>/` folder for the cluster you want
(so `VCF_NAMESPACE`/`VCF_CLUSTER` resolve correctly — see
[chapter 07](07-repo-structure-and-conventions.md)):

```
mise run context:cluster
```

Same Supervisor endpoint as above — `vcf` resolves the actual VKS cluster via
`--workload-cluster-name`/`--workload-cluster-namespace`, there's no separate cluster API URL to
look up.

## re-authenticating later

Once your session expires:

```
mise run context:refresh
```

or switch back to an already-created context for wherever you're currently standing with:

```
mise run context:use
```

See [`mise-tasks/context.toml`](../mise-tasks/context.toml) for exactly what each of these runs.
