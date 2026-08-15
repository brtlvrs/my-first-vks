# the cookbook

Read these in order the first time — each chapter assumes the previous ones. After that, treat
it as a reference and jump straight to the chapter you need.

- [00 — introduction](00-introduction.md)
- [01 — prepare your workstation](01-prepare-your-workstation.md)
- [02 — SSH keys](02-ssh-keys.md)
- [03 — required platform services](03-required-platform-services.md)
- [04 — VCF-native services](04-vcf-native-services.md)
- [05 — secret management](05-secret-management.md)
- [06 — connecting to the Supervisor](06-connecting-to-supervisor.md)
- [07 — repo structure and conventions](07-repo-structure-and-conventions.md)
- [08 — provisioning a cluster](08-provisioning-a-cluster.md)
- [09 — deploying your first app](09-deploying-your-first-app.md)
- [10 — day-2 operations](10-day2-operations.md)
- [11 — backups with Velero (optional addon)](11-backups-with-velero.md)
- [12 — troubleshooting cookbook](12-troubleshooting-cookbook.md)
- [13 — next steps](13-next-steps.md)
- [14 — infrastructure prerequisites](14-infrastructure-prerequisites.md)
- [15 — further reading](15-further-reading.md)

This is a hand-maintained index, not an auto-generated one — `mise run docs:toc` (see
[`mise-tasks/docs.toml`](../mise-tasks/docs.toml)) builds each *chapter's own* in-page table of
contents from its headings, but has no way to know about other files, so it can't maintain this
list. Update it by hand when you add, remove, or rename a chapter.

## presentation

[`presentation/slides.md`](presentation/slides.md) is a short [Marp](https://marp.app/) slide
deck version of this cookbook — a repo-tour to walk a new team through, rather than something to
read. Two ways to view it:

- `mise run docs:present` — serves it as interactive HTML with live reload at
  `http://localhost:8080` (installs `@marp-team/marp-cli` on first run). This binds to **all**
  network interfaces, not just `localhost` — from another machine on the same network, browse to
  `http://<this-machine's-LAN-IP-or-hostname>:8080` while the task is still running (it's a
  foreground process, not a background service). If that doesn't connect, check whether a host
  firewall is blocking inbound port 8080.
- `mise run docs:present-pdf` — renders it to `docs/presentation/slides.pdf` (gitignored — a
  build artifact, not something to commit) for offline viewing or sharing as a file. This needs an
  actual browser (Chrome, Edge, or Firefox) installed on the machine running it — `marp --pdf`
  doesn't bundle one. On a minimal/headless Linux install, that's often the missing piece:
  `npx @puppeteer/browsers install chrome@stable` downloads a portable Chrome, but it also needs
  a handful of shared libraries (`libatk-1.0.so.0` and similar) that a minimal server image may
  not have — install your distro's usual headless-Chrome dependency set
  (e.g. Debian/Ubuntu: `libatk-bridge2.0-0`, `libgtk-3-0`, `libnss3`, `libasound2`) if you hit
  that.
