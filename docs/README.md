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
- [11 — backups with Velero](11-backups-with-velero.md)
- [12 — troubleshooting cookbook](12-troubleshooting-cookbook.md)

This is a hand-maintained index, not an auto-generated one — `mise run docs:toc` (see
[`mise-tasks/docs.toml`](../mise-tasks/docs.toml)) builds each *chapter's own* in-page table of
contents from its headings, but has no way to know about other files, so it can't maintain this
list. Update it by hand when you add, remove, or rename a chapter.
