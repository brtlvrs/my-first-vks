# 02 — SSH keys

You'll use an SSH key to authenticate to whatever git server hosts this repo (and its
customer-specific fork). This chapter covers generating one properly and — the part that's
usually skipped and then causes daily annoyance — setting up your SSH agent so you're not typing
a passphrase on every single `git push`.

<!-- toc -->

- [generate a key](#generate-a-key)
- [use a passphrase](#use-a-passphrase)
- [stop re-entering the passphrase every time](#stop-re-entering-the-passphrase-every-time)
  * [macOS](#macos)
  * [Linux](#linux)
  * [Windows](#windows)
- [add the public key to your git server](#add-the-public-key-to-your-git-server)

<!-- tocstop -->

## generate a key

```
ssh-keygen -t ed25519 -C "you@example.com"
```

`ed25519` is the modern default — shorter keys, faster, at least as strong as RSA-4096. Accept
the default file location (`~/.ssh/id_ed25519`) unless you have a specific reason not to.

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
