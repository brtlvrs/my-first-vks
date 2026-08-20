# 01 — prepare your workstation

This chapter gets a blank workstation (Windows, macOS, or Linux) to the point where
`mise run doctor` reports everything green.

<!-- toc -->

- [1. install git](#1-install-git)
- [2. install mise](#2-install-mise)
- [3. configure mise's global settings](#3-configure-mises-global-settings)
- [4. install the pinned CLIs](#4-install-the-pinned-clis)
- [5. install the vcf CLI (manual step)](#5-install-the-vcf-cli-manual-step)
- [6. editor](#6-editor)
- [7. verify](#7-verify)
- [8. optional: a shell prompt that shows your kube context](#8-optional-a-shell-prompt-that-shows-your-kube-context)

<!-- tocstop -->

## 1. install git

**Windows:** `winget install Git.Git`. This also installs **Git Bash**, which matters here for a
reason that isn't obvious yet — see step 3.

**macOS:** `xcode-select --install`, or `brew install git`.

**Linux:** use your distro's package manager (`apt install git`, `dnf install git`, ...).

Then set your identity (used for every commit you make in any repo):

```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

[Chapter 02](02-ssh-keys.md) covers SSH keys for authenticating to a git server.

## 2. install mise

**macOS / Linux:**

```
curl -fsSL https://mise.run | sh
```

**Windows:**

```
winget install jdx.mise
```

After installing, activate mise in your shell — otherwise its shims/tools never resolve on `PATH`.
Run `mise doctor` (mise's own built-in doctor, not this repo's `mise run doctor` yet — that comes
later); if this step is missing it reports a `1 problem found` telling you to do exactly this:

**macOS / Linux (bash):**

```
echo 'eval "$(mise activate bash)"' >> ~/.bashrc
```

**macOS / Linux (zsh):**

```
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
```

**Windows (PowerShell):**

```
Add-Content -Path $PROFILE -Value 'mise activate pwsh | Out-String | Invoke-Expression'
```

Open a new shell afterward and re-run `mise doctor` — `activated: yes` confirms it worked. (For
non-interactive setups such as CI, `mise doctor` suggests adding the shims directory,
`~/.local/share/mise/shims`, to `PATH` directly instead of activating.)

## 3. configure mise's global settings

This repo's tasks are written as inline bash inside `.toml` files (see
[chapter 07](07-repo-structure-and-conventions.md)). mise's own default inline shell is a plain
POSIX `sh` — **not** bash — on every OS, and these tasks use bash-only syntax (`set -o pipefail`,
`${VAR:+...}`, etc.) that fails under `sh` with a cryptic `Illegal option` error. So this step
isn't Windows-specific busywork; skip it and tasks will fail on Linux/macOS too, just with a less
obvious symptom than on Windows (where there's no `bash` on `PATH` at all without Git Bash).

Open (or create) your global mise config file — `~/.config/mise/config.toml` on macOS/Linux, or
`<user folder>\.config\mise\config.toml` on Windows — and add:

```toml
[settings]
unix_default_inline_shell_args = "bash -c"
windows_default_inline_shell_args = '"c:/Program Files/Git/bin/bash.exe" -c'
```

Adjust the Windows path if Git was installed somewhere other than the default location. Confirm
it worked: `mise run mise:env` from the repo root should print a block of `VCF_*`/`MISE_*`
variables with no shell errors above it.

## 4. install the pinned CLIs

The root [`mise.toml`](../mise.toml) `[tools]` block pins the versions this repo expects:

| tool | what it's for |
| --- | --- |
| `kubectl` | the Kubernetes CLI — everything in `apps/` and `platform/` eventually goes through it |
| `kubectx` | switch between kubeconfig contexts (e.g. the Supervisor vs. a workload cluster) |
| `kubens` | switch the current namespace within a context |
| `k9s` | terminal UI for browsing/inspecting a cluster interactively |
| `jq` | JSON processor, used by several tasks that parse `kubectl -o json` output |
| `yq` | the YAML equivalent of `jq`, used for reading/editing manifests from tasks |
| `kustomize` | the `base`/`overlay`/`component` engine behind every manifest in `apps/` and `platform/` |
| `fzf` | fuzzy picker — powers the `cluster:delete` interactive picker (see [chapter 10](10-day2-operations.md#deleting-a-cluster)), and `kubectx`/`kubens` pick it up automatically for interactive context/namespace switching once it's on `PATH` |

From the repo root, just run:

```
mise install
```

mise reads `[tools]` and installs everything listed, at the pinned versions. You don't need to
install these globally or manage them by hand — mise makes them available whenever your shell is
inside this repo. For what each tool actually is and links to its own docs, see
[chapter 15](15-further-reading.md#tools-this-repo-uses-directly).

## 5. install the vcf CLI (manual step)

`vcf` is Broadcom's CLI for talking to a VCF Supervisor. It isn't distributed through any package
manager mise can reach — it ships **from the Supervisor itself**:

1. Open `https://<your Supervisor endpoint>/` in a browser (see
   [chapter 06](06-connecting-to-supervisor.md) for where this value comes from).
2. Download the CLI build for your OS.
3. **Windows:** unzip it, rename the executable to `vcf.exe`, and place it under
   `C:\Program Files\vcf-cli\` (or anywhere already on your `PATH`).
   **macOS/Linux:** unzip it and place the `vcf` binary somewhere on your `PATH`
   (e.g. `/usr/local/bin`).
4. Confirm it resolves: `vcf version`.

## 6. editor

Visual Studio Code is the recommended editor — install the **YAML** and **Kubernetes**
extensions for inline validation of the manifests in `apps/` and `platform/`.

## 7. verify

From the repo root:

```
mise run doctor
```

This checks every CLI above and reports `OK`/`WARN`/`FAIL` per tool (see
[`mise-tasks/doctor.toml`](../mise-tasks/doctor.toml)). Fix anything reported as `FAIL` before
moving on — `WARN` (e.g. `vcf` or `fzf` not yet installed) is fine to defer if you're not ready
for that step yet.

## 8. optional: a shell prompt that shows your kube context

Not checked by `mise run doctor` — purely a personal-comfort suggestion. When you're jumping
between the Supervisor and workload-cluster contexts with `kubectx`/`kubens` (step 4), it's easy
to lose track of which one is currently active. [starship](https://starship.rs/) is a cross-shell
prompt that can show it for you.

**macOS/Linux:**

```
curl -sS https://starship.rs/install.sh | sh
```

(or `brew install starship` on macOS)

**Windows:** `winget install --id Starship.Starship`

Then activate it, the same way you activated mise in step 2:

**bash:** `echo 'eval "$(starship init bash)"' >> ~/.bashrc`

**zsh:** `echo 'eval "$(starship init zsh)"' >> ~/.zshrc`

**PowerShell:** `Add-Content -Path $PROFILE -Value 'Invoke-Expression (&starship init powershell)'`

Open a new shell to pick it up. One catch: starship's `kubernetes` module — the piece that prints
the current context/namespace — is **disabled by default**. Enable it in starship's own config
(`~/.config/starship.toml` on macOS/Linux, `%APPDATA%\starship\config.toml` on Windows):

```toml
[kubernetes]
disabled = false
format = '[$symbol$context( \($namespace\))]($style) '
```

See [starship.rs/config/#kubernetes](https://starship.rs/config/#kubernetes) for the full set of
options (aliasing long context names, styling, etc.).
