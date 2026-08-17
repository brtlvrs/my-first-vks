# Open items

These surfaced while building this repo out from the `vks-apps`/`vks-platform` patterns and need a decision or verification before they can be called final. See `docs/` for what's already documented.

1. **Per-OS ssh-agent guidance in `docs/02-ssh-keys.md`.** Checked against current published best-practice sources (key expiry via `ssh-add -t`, `ProxyJump` over agent forwarding, unused-key hygiene — all now added) — but still not verified against a specific real customer's actual OS fleet. Review against what the real target team uses before treating it as fully final.
2. **vSphere Pods (`apps/hello-vsphere-pod/`) use case.** VM Service now has a documented candidate use case and a "declarative ≠ secure" framing written up in `apps/README.md#when-to-reach-for-vm-service` — vSphere Pods' own "when would you actually reach for this over a regular VKS-cluster app" is still open. Revisit once there's a concrete candidate workload.
3. **Promote to `v1.0.0`.** Currently `v0.1.0` — per SemVer, `0.y.z` means "still initial development, anything may change." Promote once items 1 and 2 above are resolved and this repo has actually been used to onboard a real customer team, not before.
