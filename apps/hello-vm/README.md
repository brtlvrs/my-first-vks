# hello-vm

A VM Service worked example: a rootless-Podman host, provisioned declaratively as a
`VirtualMachine` custom resource straight into a vSphere Namespace. See
[`../../docs/00-introduction.md`](../../docs/00-introduction.md) for what VM Service is, and
[`../README.md#vm-service-and-vsphere-pods`](../README.md#vm-service-and-vsphere-pods) for how
this differs structurally from a container-based app.

<!-- toc -->

- [deploying](#deploying)
- [what persists across a redeploy, and what doesn't](#what-persists-across-a-redeploy-and-what-doesnt)
- [running the demo compose stack](#running-the-demo-compose-stack)
- [reaching it from outside: the loadbalancer components](#reaching-it-from-outside-the-loadbalancer-components)
- [vm mise tasks](#vm-mise-tasks)
- [CHANGE_ME placeholders](#change_me-placeholders)

<!-- tocstop -->

## deploying

1. Fill in the `CHANGE_ME_*` placeholders — see the table at the bottom of this file.
2. Create the login-password Secret (never commit the hash):
   `mise run vm:set-password` from `overlays/example-namespace/`, or manually:
   `kubectl create secret generic hello-vm-passwd -n <namespace> --from-literal=password="$(openssl passwd -6)"`.
3. From `overlays/example-namespace/`: `mise run app:render` to sanity-check, then
   `mise run context:supervisor` (or `context:use`) and `mise run vm:redeploy` (first-time deploy
   and later redeploys both go through this same task — see [chapter 08](../../docs/08-provisioning-a-cluster.md)
   for the general pattern this follows).
4. Once it reports an IP: `ssh hello@<ip>` (the password you set in step 2, or your SSH key).

cloud-init needs a few minutes after the VM reports an IP to finish package installs and disk
setup before SSH is fully usable.

## what persists across a redeploy, and what doesn't

Same two-disk pattern as the production repo this is modeled on — a disposable OS disk, plus a
`hello-vm-data` PVC created independently of the VM so it survives `kubectl delete virtualmachine`:

| Disk | Mount | Survives redeploy? |
|------|-------|---------------------|
| OS (from `imageName`) | `/` | No — rebuilt from image + cloud-init every time |
| Data (`hello-vm-data` PVC) | `/var/lib/hello-vm`, bind-mounted onto `/home/hello` and `/root` | Yes |

Podman's rootless storage root (`$HOME/.local/share/containers`) lives under `/home/hello`, so it
persists as part of that same bind mount — no separate volume needed. **Put your compose project
(the compose file, `.env`, any bind-mount targets it references) under `/home/hello` or `/root`**
— anything outside those two paths (or `/var/lib/hello-vm` directly) is lost on redeploy.

The actual data-loss boundary is `kubectl delete pvc hello-vm-data -n <namespace>` — never run
that unless the intent is to permanently wipe state.

Need more room on `hello-vm-data` later? Resizing a VM Service app's data disk needs one extra
manual step beyond a normal PVC resize — see
[chapter 10](../../docs/10-day2-operations.md#resizing-a-vm-service-apps-data-disk).

## running the demo compose stack

[`../compose/docker-compose.yaml`](../compose/docker-compose.yaml) isn't applied by
kustomize/mise — there's no automated delivery mechanism for it here, same as the repo this is
modeled on. Once you can SSH in:

```
scp ../compose/docker-compose.yaml hello@<vm-ip>:~/docker-compose.yaml
ssh hello@<vm-ip> 'docker-compose -f ~/docker-compose.yaml up -d'
```

`hello` (the `hello-world` image) is a trivial smoke test — it prints one line and exits,
confirming Podman/docker-compose actually works. `it-tools` gives you something with a web UI to
look at — see the next section for how to reach it. Both land under `/home/hello`, so they (and
any data volumes they create) survive a redeploy per the table above.

## reaching it from outside: the loadbalancer components

`overlays/example-namespace/kustomization.yaml` mixes in two
[kustomize components](https://kubectl.docs.kubernetes.io/guides/config_management/components/),
each rendering a `VirtualMachineService` — vm-operator's own `LoadBalancer`-type CRD, **not** a
plain core `v1.Service` (a VM Service VM isn't a Pod, so a normal Service's endpoint selection
doesn't apply the same way):

- `../../components/http-loadbalancer/` — exposes the compose stack's `it-tools` UI, port 80 →
  the VM's 8080.
- `../../components/ssh-loadbalancer/` — exposes SSH, port 22 → the VM's 22.

They're **separate components on purpose**, not one combined Service — so you can expose the app
publicly while keeping SSH reachable only from inside the network (or the other way around),
toggled independently in `overlays/example-namespace/kustomization.yaml`'s `components:` list
without touching the other. Both are enabled by default here so the worked example is fully
reachable out of the box; delete either line to turn that one off.

Once deployed:

```
kubectl get virtualmachineservice -n <namespace>
```

lists both, each with its own external IP once your Supervisor's load-balancer provider
(NSX ALB, NSX-LB, or FLB — see [chapter 14](../../docs/14-infrastructure-prerequisites.md))
assigns one. Browse to `http://<http-loadbalancer-ip>/` for `it-tools`, and
`ssh hello@<ssh-loadbalancer-ip>` instead of the VM's own `primaryIP4` if you'd rather not depend
on being on the same network segment as the VM itself.

## vm mise tasks

Defined in [`../mise-tasks/vm.toml`](../mise-tasks/vm.toml), run from
`overlays/example-namespace/`:

| Task | Does |
|------|------|
| `vm:power-on` / `vm:power-off` | Power on / hard power off |
| `vm:restart` | vm-operator-managed restart (`nextRestartTime: now`) |
| `vm:delete` | **Destructive.** Deletes the VM (typed-name confirmation) — the data-disk PVC survives |
| `vm:redeploy` | First-time deploy, or delete-then-reapply if it already exists — waits for an IP |
| `vm:set-password` | Create/rotate the login-password Secret |
| `pvc:sync-size` | Sync `base/pvc.yaml`'s storage size to the bound PV's real size after an out-of-git resize — see [chapter 10](../../docs/10-day2-operations.md#resizing-a-vm-service-apps-data-disk) |

## CHANGE_ME placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `CHANGE_ME_NAMESPACE` | `base/vm.yaml`, `base/pvc.yaml` | overridden automatically by the overlay's `namespace:` field — no manual edit needed |
| `CHANGE_ME_VM_CLASS` | `base/vm.yaml` | `kubectl get virtualmachineclass` |
| `CHANGE_ME_VM_IMAGE` | `base/vm.yaml` | `kubectl get virtualmachineimage` |
| `CHANGE_ME_STORAGE_CLASS` | `base/vm.yaml`, `base/pvc.yaml` | `kubectl get storageclass` |
| `CHANGE_ME_SSH_PUBLIC_KEY` | `base/vm.yaml` | your public key, see [`../../docs/02-ssh-keys.md`](../../docs/02-ssh-keys.md) |
| `example-namespace` (folder + `VCF_NAMESPACE`) | `overlays/example-namespace/mise.toml` | your real vSphere Namespace |
