# 12 — troubleshooting cookbook

Common failure modes and what they usually mean.

<!-- toc -->

- ["context not found" when running a mise task](#context-not-found-when-running-a-mise-task)
- ["Missing VCF_NAMESPACE env var" (or similar)](#missing-vcf_namespace-env-var-or-similar)
- [`sh: 1: set: Illegal option -o pipefail` (any OS)](#sh-1-set-illegal-option--o-pipefail-any-os)
- [mise task fails on Windows with no useful error](#mise-task-fails-on-windows-with-no-useful-error)
- [app pod stuck Pending or refused by admission](#app-pod-stuck-pending-or-refused-by-admission)
- [copied an overlay folder and things target the wrong namespace](#copied-an-overlay-folder-and-things-target-the-wrong-namespace)
- [`cluster:delete` has no interactive picker](#clusterdelete-has-no-interactive-picker)
- [`kubectl debug node` pod won't schedule, or lingers afterward](#kubectl-debug-node-pod-wont-schedule-or-lingers-afterward)
- [PVC resize stuck at `FileSystemResizePending`](#pvc-resize-stuck-at-filesystemresizepending)
- [git line-ending changes on every checkout (Windows)](#git-line-ending-changes-on-every-checkout-windows)

<!-- tocstop -->

## "context not found" when running a mise task

You haven't authenticated yet, or your session expired. Run `mise run context:supervisor` (first
time) or `mise run context:refresh` / `context:use` (already created before) — see
[chapter 06](06-connecting-to-supervisor.md).

## "Missing VCF_NAMESPACE env var" (or similar)

Every `cluster:*`/`context:*` task guards on the env vars it needs and tells you exactly which
one is missing. This almost always means you're either not standing in the folder you think you
are, or that folder (or a parent of it) is missing its `mise.toml` — run `mise run mise:env` from
where you are to see exactly what mise has resolved, and compare against
[chapter 07](07-repo-structure-and-conventions.md#how-mise-env-cascades).

## `sh: 1: set: Illegal option -o pipefail` (any OS)

mise's own default inline shell is a plain POSIX `sh`, not bash, on every OS — this repo's tasks
are bash. You skipped (or mis-typed) the global mise settings block from
[chapter 01, step 3](01-prepare-your-workstation.md#3-configure-mises-global-settings). This bites
Linux and macOS too, not just Windows — add `unix_default_inline_shell_args = "bash -c"` to your
global `~/.config/mise/config.toml` and re-run.

## mise task fails on Windows with no useful error

Almost always the global mise shell setting from
[chapter 01, step 3](01-prepare-your-workstation.md#3-configure-mises-global-settings) is missing
or points at the wrong Git Bash path. Confirm Git Bash exists at the path in your
`~/.config/mise/config.toml` (or `<user folder>\.config\mise\config.toml`), and that you installed
Git via `winget install Git.Git` (not some other distribution that puts `bash.exe` somewhere
else).

## app pod stuck Pending or refused by admission

If the namespace enforces the Restricted Pod Security Standard (`apps/hello-vks/`,
`hello-vsphere-pod/`, and `it-tools/`'s VKS overlay do; `it-tools/` itself runs at Baseline, see
[chapter 10](10-day2-operations.md#pod-security-standards-understanding-admission)), a container
that runs as root, doesn't drop all capabilities, or doesn't set `seccompProfile` gets **rejected
outright at `kubectl apply` time**, not scheduled-then-crash-looping — that's the tell: no pod
ever gets created at all. `kubectl describe pod <pod> -n <namespace>` or
`kubectl get events -n <namespace>` shows an admission-webhook denial message naming the exact
field that failed. See [chapter 10](10-day2-operations.md#pod-security-standards-understanding-admission)
for what each PSS level actually requires and why `it-tools/` deliberately runs at Baseline
instead of Restricted — compare your own container's `securityContext` against whichever of
`apps/hello-vks/base/deployment.yaml` (Restricted-compliant) or `apps/it-tools/base/deployment.yaml`
(Baseline, root + one narrow capability) is the closer match for what your image actually needs.

## copied an overlay folder and things target the wrong namespace

A copy-pasted overlay has **two** places that need updating to match, not one — it's easy to
update only one and get a confusing mismatch:

1. The `mise.toml`'s `VCF_NAMESPACE` (and `VCF_CLUSTER`, if present) — controls which Supervisor
   context `mise run context:*` and `cluster:*`/`app:*` tasks use.
2. The `kustomization.yaml`/`kustomization.yml`'s `namespace:` field — controls what namespace the
   *rendered manifest itself* declares.

If these disagree, `app:render`/`cluster:render` will look fine, but `app:apply`/`cluster:apply`
will land the resource somewhere other than where your `mise` context is pointed, and later
`kubectl get` commands against "the right" namespace won't find it.

## `cluster:delete` has no interactive picker

`fzf` isn't installed — `mise run doctor` reports this as a `WARN`, not a `FAIL`, since it's
optional. Either install `fzf`, or pass the cluster name directly:
`mise run cluster:delete <name>`. See [`TODO.md`](../TODO.md).

## `kubectl debug node` pod won't schedule, or lingers afterward

Two separate gotchas with [chapter 10's node-shell walkthrough](10-day2-operations.md#accessing-a-nodes-shell-without-ssh):

- **Refused/stuck Pending** — you ran it against an application namespace instead of
  `kube-system`. A privileged debug pod can't schedule under the Restricted Pod Security Standard
  that `hello-vks`-style namespaces enforce; the `-n kube-system` in the command is load-bearing,
  not incidental.
- **Old debug pods piling up** — `kubectl debug node` doesn't clean up after itself. If
  `kubectl get pod -n kube-system --context <namespace>:<cluster-name>` shows several
  `node-debugger-*` pods sitting around from past sessions, delete the ones you're done with;
  they don't expire on their own.

## PVC resize stuck at `FileSystemResizePending`

Expected for any VM Service app's data disk (`hello-vm-data` and friends in `apps/hello-vm/`) —
these VMs aren't pods, so there's no kubelet around to finish the node-side half of a volume
expansion the way a normal PVC-in-a-pod resize would. The backing disk itself has already grown
(`kubectl get pv` shows the new size); you just need to trigger the node-side resize yourself with
a VM restart and an in-guest `resize2fs`/`xfs_growfs`. The PVC's reported size and this condition
can then stay stale indefinitely afterward — that's cosmetic, not a sign anything's still broken.
Full procedure: [chapter 10](10-day2-operations.md#resizing-a-vm-service-apps-data-disk).

## git line-ending changes on every checkout (Windows)

If `git status` shows every file as modified immediately after a fresh clone with no edits made,
Windows' default CRLF line-ending conversion is fighting with this repo's shell scripts (mise
`.toml` task files with inline bash need LF, not CRLF, to run correctly under Git Bash). Set, once,
globally:

```
git config --global core.autocrlf input
```

then re-clone (or `git rm --cached -r .` followed by `git checkout .` in an existing clone) so the
working tree gets re-normalized.
