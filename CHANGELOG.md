# Changelog

All notable changes to this repo are documented here, in the
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format. Versioning follows
[Semantic Versioning](https://semver.org/) — see
[`docs/13-next-steps.md#versioning-this-repo`](docs/13-next-steps.md#versioning-this-repo) for
what a major/minor/patch bump means specifically for a repo you fork rather than install as a
package.

While the major version is `0`, per the SemVer spec anything may still change — this repo hasn't
yet been used for a real customer onboarding, and `TODO.md` still has open design questions.

## [Unreleased]

### Added

- [`LICENSE`](LICENSE) (MIT), plus a `README.md` "disclaimer" section spelling out what "AS IS, no
  warranty" means in practice for a repo that drives real vSphere/VCF infrastructure.
- An ASCII art banner ([`BANNER.txt`](BANNER.txt)) at the top of `README.md`.
- Chapter 12: a quick-diagnosis entry for `kubectl debug node` pods that won't schedule (wrong
  namespace/PSS level) or that pile up after use, pointing at the existing walkthrough in
  [chapter 10](docs/10-day2-operations.md#accessing-a-nodes-shell-without-ssh).

### Changed

- Chapter 02: "generate a key" is now broken out per macOS/Linux/Windows (previously one generic
  command), including the Windows-specific `icacls` fix for OpenSSH's "permissions are too open"
  error.

## [0.2.1] - 2026-08-17

### Fixed

- `govc:login`/`govc:logout` fall back to an insecure connection (`GOVC_INSECURE=true`, with a
  warning) when `vcsa-ca.pem` isn't present yet, instead of failing outright.

## [0.2.0] - 2026-08-17

### Added

- `mise run govc:login` / `govc:logout` — optional tasks for direct `govc` access to vCenter
  (outside what `kubectl`/`vcf` expose), documented in
  [chapter 14](docs/14-infrastructure-prerequisites.md#govc-optional-direct-vcenter-access).
  Adds `GOVC_URL` to the root `mise.toml [env]` block.

## [0.1.0] - 2026-08-17

### Added

- The full `docs/` cookbook (chapters 00-15): workstation setup, SSH keys, required platform
  services, VCF-native services, secret management, connecting to the Supervisor, repo
  conventions, provisioning a cluster, deploying an app, day-2 operations, Pod Security
  Standards, troubleshooting, next steps, infrastructure prerequisites for a VI/NSX admin, and a
  further-reading index.
- `platform/` — namespace-first kustomize layout with a worked `example-namespace/` provisioning
  a VKS cluster, plus an optional (not shipped by default) Velero backups addon described in
  chapter 11.
- `apps/` — component-first kustomize layout with worked examples: `hello-vks` (a Restricted-PSS
  compliant container), `hello-vsphere-pod` (the same container as a native vSphere Pod),
  `hello-vm` (VM Service, with a persistent data disk across redeploys), and `it-tools` (a real
  app deployed both as a VKS workload and as a vSphere Pod).
- `mise.toml` / `mise-tasks/` — pinned CLI versions (`kubectl`, `kubectx`, `kubens`, `k9s`, `jq`,
  `yq`, `kustomize`, `fzf`), a `doctor` task to validate a workstation, and shared tasks for
  context-switching, docs maintenance, and the presentation deck below.
- `docs/presentation/slides.md` — a Marp slide deck version of the cookbook, servable as HTML
  (`mise run docs:present`) or rendered to PDF (`mise run docs:present-pdf`).
- `TODO.md` — open design questions tracked in the open rather than resolved speculatively.
