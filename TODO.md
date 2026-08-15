# Open items

These surfaced while building this repo out from the `vks-apps`/`vks-platform` patterns and need a decision or verification before they can be called final. See `docs/` for what's already documented.

1. **`fzf` as a dependency.** `platform/mise-tasks/cluster.toml`'s `cluster:delete` (ported from `vks-platform`) uses an `fzf` picker when no cluster name is given. `fzf` is *not* pinned in the root `mise.toml [tools]` and `mise run doctor` only warns if it's missing. Decide: pin it as a hard requirement, or keep every interactive task working without it (plain prompt fallback).
2. **Per-OS ssh-agent guidance in `docs/02-ssh-keys.md`.** Drafted for macOS/Windows/Linux from general best practice, not from a specific customer's actual fleet — review against what the real target team uses before treating it as final.
3. **`platform/bases/velero/veleroservice.yaml` plugin image references** (`velero-plugin-for-aws`/`velero-plugin-for-vsphere` versions) are carried over from `vks-platform`, which itself flagged them as read off a live Supervisor's default config but not confirmed end-to-end against a real `velero-vsphere install`. Re-verify against the target customer's actual Supervisor/Velero operator version.
