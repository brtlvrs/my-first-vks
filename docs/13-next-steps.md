# 13 — next steps

Everything in chapters 00-12 is deliberately simple: one repo, manual `mise run *:apply`,
copy-and-fill-in overlays. That's the right starting point for a team new to `kubectl`/VKS — but
it's a starting point, not an end state. This chapter is a roadmap of directions to grow into
once the basics feel routine. **Nothing here is implemented in this repo** — no new folders, no
new tasks — it's advisory, for when you're ready to act on it.

<!-- toc -->

- [from manual apply to GitOps](#from-manual-apply-to-gitops)
- [platform as a product](#platform-as-a-product)
- [versioning this repo](#versioning-this-repo)
- [12-factor apps](#12-factor-apps)
- [building broader team fundamentals](#building-broader-team-fundamentals)
- [where to track these](#where-to-track-these)

<!-- tocstop -->

## from manual apply to GitOps

Right now, "deploying" means a person runs `mise run cluster:apply` or `app:apply` from their own
workstation. **GitOps** flips this around: a controller running inside the cluster (Flux or
ArgoCD are the two mainstream choices) continuously reconciles the live cluster state against
whatever is committed in git — merging a change to `main` *is* the deploy action, not a CLI
command someone remembered to run.

What that buys you: a full audit trail (git history is deploy history — who changed what, when,
reviewed by whom, if you require PRs), and drift correction (someone runs an emergency
`kubectl edit` by hand? the controller quietly reverts it back to what git says should be
running, or flags the drift). What it costs: another thing to operate (the GitOps controller
itself needs installing, upgrading, and monitoring) and a habit change for the team (PR-and-merge
instead of "I ran the apply command from my laptop"). Before self-hosting Flux or ArgoCD, check
[chapter 04](04-vcf-native-services.md) — whether something equivalent is already offered as a
Supervisor Service is worth ten minutes of checking before you commit to operating it yourself.

## platform as a product

A pattern worth knowing about once you're running more than one or two vSphere Namespaces off
this repo: split `platform/bases/` out of this repo entirely, into its own **product repo**,
released via git tags (`v1.0.0`, `v1.1.0`, ...). Each namespace's overlay, living in a separate
**consumer repo** (or consumer folder), then references a *specific tagged version* of the
product repo via kustomize's remote-base syntax instead of a local relative path:

```yaml
# instead of: resources: [../../bases/vks-cluster]
resources:
  - git@your-git-host:your-org/platform-product.git//vks-cluster?ref=v1.2.0
```

This is what actually gets you independent version pinning per environment — a sandbox vSphere
Namespace's overlay can point at `v2.0.0-rc1` while production points at `v1.4.0`, and "promoting"
sandbox's validated changes to production becomes "bump the `?ref=` in production's overlay,"
nothing more. It pairs naturally with the GitOps section above: a product upgrade becomes a
one-line PR that a controller then rolls out for you, giving you a real, git-reviewable promotion
pipeline instead of someone remembering which cluster is on which version.

**Why this repo doesn't do it today:** it's a genuine jump in operational weight — a second repo
with its own release/tagging discipline, and `kustomize build` now needs live git/network access
to resolve a remote ref instead of just reading a local folder. That's the wrong trade for a repo
whose whole point is flattening the learning curve. When you do reach for it, the migration itself
is mechanical: move `platform/bases/` into a new repo, tag it, push it, then update each
namespace's `resources:` entries to the remote-ref form above.

## versioning this repo

This repo itself — not just the "platform as a product" idea above — is tagged using
[Semantic Versioning](https://semver.org/): `vMAJOR.MINOR.PATCH`. The point of that scheme,
specifically for a repo teams fork and later re-point at a newer tag, is that **the tag name
alone answers "is pulling this update safe for my fork,"** with no need to read anything first:

- **major** — a breaking change to something a fork would depend on: a renamed/restructured
  folder, a renamed or removed mise task, a changed `CHANGE_ME_*` convention. Stop and read
  [`CHANGELOG.md`](../CHANGELOG.md) before re-pointing your fork.
- **minor** — something added without breaking what's already there: a new chapter, a new worked
  example under `apps/`, a new task. Safe to pull.
- **patch** — docs/typo fixes, no structural change at all. Always safe to pull.

While the major version is `0` (`v0.y.z`), per the SemVer spec itself anything may still change —
this repo hasn't yet been used for a real customer onboarding. See [`TODO.md`](../TODO.md) for
what's gating a `v1.0.0`.

**Cutting a release:**

1. Move the entries accumulated under `## [Unreleased]` in `CHANGELOG.md` under a new
   `## [X.Y.Z] - YYYY-MM-DD` header, choosing the bump per the rules above.
2. Update the "Current version" line and link at the top of [`README.md`](../README.md) to match.
3. `mise run release:tag` — reads that new header out of `CHANGELOG.md` and creates a local
   annotated tag `vX.Y.Z`. It does not push anything.
4. Push the tag yourself once you're ready: `git push <remote> vX.Y.Z`, once per remote.

## 12-factor apps

The [12-factor app](https://12factor.net/) methodology is a widely-used set of conventions for
building applications that run well on platforms like this one. `apps/hello-vks` already leans on
a few of them, worth calling out explicitly so you carry the habit forward into your own apps:

- **III. Config in the environment** — this repo's `CHANGE_ME`/`mise.toml [env]` cascading
  ([chapter 07](07-repo-structure-and-conventions.md#how-mise-env-cascades)) keeps
  environment-specific values out of the base manifest. The gap: that's config for *this repo's
  tooling*, not necessarily config the running application itself reads at startup — a genuinely
  12-factor app reads its own config from environment variables (via a `ConfigMap`/`Secret`
  wired into the container's `env:`), rather than having different values baked into different
  overlays' patched YAML.
- **IV. Backing services as attached resources** — an app shouldn't care whether the S3 bucket or
  git server it talks to is self-hosted (chapter 03) or VCF-native (chapter 04); swapping one for
  the other should be a config change, not a code change.
- **V. Strict separation of build, release, and run** — CI builds an image; a kustomize overlay
  (assembling that image reference with this environment's config) is your "release"; `kubectl
  apply` is "run." Keeping these distinct is most of what this repo's `base/`+`overlay` structure
  already buys you.
- **IX. Disposability** — fast startup, graceful shutdown, no reliance on local disk state.
  `hello-vks` has no persistent volumes and sensible resource requests/limits for exactly this
  reason — a pod should be safe to kill and reschedule at any time.

Read [12factor.net](https://12factor.net/) directly for the full list (there are twelve, this
only calls out the ones most visible in this repo already) — it's short and worth reading in full
once, not just referencing.

## building broader team fundamentals

This repo deliberately covers only the VKS-specific slice: kustomize, mise, and the Supervisor.
If some of your team is newer to Linux, containers, or DevOps practice more broadly, a structured
self-paced curriculum is a good parallel track — search GitHub for **"90DaysOfDevOps"**, a
well-known free, day-by-day program covering Linux, git, Docker, Kubernetes, cloud fundamentals,
and scripting from the ground up. It's a complement to this repo, not a prerequisite — nothing in
chapters 00-12 requires finishing it first.

## where to track these

[`TODO.md`](../TODO.md) is the near-term list — small, concrete items blocking this repo from
being "done." This chapter is the longer-horizon list: none of chapters 00-12 depend on anything
above, they're what you reach for once the basics feel routine, not before.
