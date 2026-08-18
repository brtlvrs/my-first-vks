# 02 — SSH keys

You'll use an SSH key to authenticate to whatever git server hosts this repo (and its
customer-specific fork). This chapter covers generating one properly and — the part that's
usually skipped and then causes daily annoyance — setting up your SSH agent so you're not typing
a passphrase on every single `git push`.

<!-- toc -->

- [generate a key](#generate-a-key)
  * [macOS](#macos)
  * [Linux](#linux)
  * [Windows](#windows)
- [use a passphrase](#use-a-passphrase)
- [stop re-entering the passphrase every time](#stop-re-entering-the-passphrase-every-time)
  * [macOS](#macos-1)
  * [Linux](#linux-1)
  * [Windows](#windows-1)
- [add the public key to your git server](#add-the-public-key-to-your-git-server)
- [optional extra hardening](#optional-extra-hardening)

<!-- tocstop -->

## generate a key

The command itself is identical on every OS — `ed25519` is the modern default (shorter keys,
faster, at least as strong as RSA-4096) and every platform below already has `ssh-keygen` on
`PATH` once [chapter 01](01-prepare-your-workstation.md) is done. Accept the default file
location it suggests (`~/.ssh/id_ed25519`) unless you have a specific reason not to. What
differs per OS is *where that ends up* and a couple of platform-specific gotchas.

### macOS

```
ssh-keygen -t ed25519 -C "you@example.com"
```

Comes with the Xcode Command Line Tools chapter 01 already has you install — nothing extra to
set up. Lands at `~/.ssh/id_ed25519`.

### Linux

```
ssh-keygen -t ed25519 -C "you@example.com"
```

Most desktop distros already have this; a minimal/server install might not. If `ssh-keygen: command
not found`, install the client package first: `sudo apt install openssh-client` (Debian/Ubuntu) or
`sudo dnf install openssh-clients` (Fedora/RHEL). Lands at `~/.ssh/id_ed25519`.

### Windows

Run it from **Git Bash** (installed in chapter 01 alongside git) so the rest of this chapter's
commands work unmodified — PowerShell's built-in OpenSSH client works too, but Git Bash keeps
you on the same `~/.ssh/...`-style paths used everywhere else in this cookbook:

```
ssh-keygen -t ed25519 -C "you@example.com"
```

This lands at `%USERPROFILE%\.ssh\id_ed25519` — the same location Git Bash's `~/.ssh/id_ed25519`
resolves to, just written the Windows way for when you need it in PowerShell (e.g. the agent
commands below). One Windows-specific gotcha to know about upfront: if OpenSSH ever refuses your
key with an `UNPROTECTED PRIVATE KEY FILE` / "permissions ... are too open" error, it means the
key file's NTFS permissions grant access beyond just your account (common if it was copied in
rather than generated in place). Fix it from PowerShell:

```powershell
icacls $env:USERPROFILE\.ssh\id_ed25519 /inheritance:r
icacls $env:USERPROFILE\.ssh\id_ed25519 /grant:r "$($env:USERNAME):F"
```

That strips inherited permissions and grants access to only your own account.

## use a passphrase

When prompted, **set a passphrase**. This is not optional best practice you can skip for
convenience — an unencrypted private key on disk means anyone who copies the file (a stolen
laptop, a misconfigured backup, a compromised account) has your identity, permanently, with no
second factor. A passphrase means the key file alone isn't enough.

The rest of this chapter exists precisely to answer "but I don't want to type that constantly" —
that's what an **SSH agent** is for: unlock the key once per login session, and every subsequent
`git`/`ssh` call re-uses that unlocked copy in memory instead of asking again.

## stop re-entering the passphrase every time

### macOS

macOS's `ssh-agent` starts automatically. Load the key into it once, using the Keychain to
remember the passphrase across reboots:

```
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

Then add this to `~/.ssh/config` (create the file if it doesn't exist) so it happens
automatically from now on, for every session:

```
Host *
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

### Linux

Most distros with a graphical login (GNOME, KDE) already run an agent and unlock it with your
login password via `gnome-keyring`/`ksshaskpass` — try `ssh-add -l` first; if it lists your key,
you're already done.

If not, start an agent from your shell profile (`~/.bashrc` / `~/.zshrc`) and add the key once:

```bash
# ~/.bashrc or ~/.zshrc
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null
fi
```

```
ssh-add ~/.ssh/id_ed25519
```

This still asks for the passphrase once per new shell session's agent (not once per `git`
command, which is the actual annoyance being solved). For "once per login, full stop," install
`keychain` (`apt install keychain` / `dnf install keychain`) and use it instead of the raw
`eval "$(ssh-agent -s)"` line — it reuses a single agent across all your terminal sessions.

### Windows

Windows ships its own OpenSSH agent as a background service — enable and start it once
(PowerShell, as Administrator):

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

Then add your key:

```powershell
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

Because it's a real Windows service (not something started per-shell), this persists across
reboots and works the same from PowerShell, Git Bash, or any other terminal.

## add the public key to your git server

`cat ~/.ssh/id_ed25519.pub` and paste the output into your git server's SSH-keys settings page.
The exact location varies by server — see [chapter 03](03-required-platform-services.md) for the
Gitea example. Never paste the *private* key (`id_ed25519`, no `.pub` suffix) anywhere.

## optional extra hardening

- **Auto-expire the unlocked key** — `ssh-add -t 3600 ~/.ssh/id_ed25519` unlocks it for 1 hour
  instead of the whole login session, so a stolen unlocked laptop is a smaller window of exposure.
  Works on macOS/Linux; **Windows' built-in OpenSSH `ssh-agent` does not support `-t`**, so this
  one's platform-limited.
- **Prefer `ProxyJump` over agent forwarding** if you ever need to hop through a bastion/jump host
  to reach something else (e.g. an `apps/hello-vm/` VM that's only reachable from inside the
  network) — agent forwarding exposes your unlocked key to whatever you forward it to, whereas
  `ProxyJump` (`ssh -J bastion target`, or a `ProxyJump` line in `~/.ssh/config`) just relays the
  connection through the bastion without ever handing it your key.
- **Delete keys you've stopped using** — from disk and from `ssh-add -l`'s list. A key that's
  gone can't be used, forwarded, or leaked; one that's still lying around a year later can.

Sources: [goteleport.com — 5 SSH Agent Best Practices](https://goteleport.com/blog/how-to-use-ssh-agent-safely/),
[smallstep.com — SSH Agent Explained](https://smallstep.com/blog/ssh-agent-explained/),
[PowerShell/Win32-OpenSSH wiki — file permissions](https://github.com/PowerShell/Win32-OpenSSH/wiki/Security-protection-of-various-files-in-Win32-OpenSSH)
(the `icacls` fix above is the community-standard fix for the "permissions are too open" error;
the wiki documents the underlying rule, not that exact command).
