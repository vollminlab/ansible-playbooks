# ansible-playbooks

Ansible automation for the Vollminlab homelab. Runs from `ansible01.vollminlab.com`.

## Layout

```
inventory/
  hosts.ini                       # all managed hosts, grouped by role
playbooks/
  k8s-upgrade.yml                 # Kubernetes minor-version upgrade, one hop at a time
  harden-cp-probes.yml            # widen apiserver/etcd probes + KCM/scheduler leader-election (anti-cascade)
  os-patch.yml                    # rolling apt full-upgrade + reboot-if-required (general OS patcher)
  cp-containerd-upgrade.yml       # control-plane containerd 1.7 -> 2.2 upgrade
  containerd-dockerhub-mirror.yml # route docker.io through the Harbor pull-through cache
  kubelet-vm-lt-mount-ordering.yml # k8sworker01: order kubelet after the /mnt/vm-lt cold-tier mount
ansible.cfg                       # default inventory, SSH key, connection settings
```

## Prerequisites

ansible01 must have:
- Ansible (`ansible --version`)
- kubectl + `~/.kube/config` pointing at the cluster
- `~/.ssh/ansible_k8s_ed25519` — the dedicated Ansible SSH key (authorized on all nodes)

## Vault password — required for every playbook

Each node's `ansible_become_password` is stored as an inline `!vault`-encrypted
value in `inventory/host_vars/<host>/vars.yml`. Every playbook here uses
`become: true`, so **all runs must supply the vault password** or they fail at
"Gathering Facts" with `Attempting to decrypt but no vault secrets found`.

Append `--ask-vault-pass` to prompt for it interactively:

```bash
ansible-playbook playbooks/<playbook>.yml --ask-vault-pass
```

Or point at a vault password file (e.g. `--vault-password-file ~/.vault_pass`,
or set `ANSIBLE_VAULT_PASSWORD_FILE`).

## Kubernetes upgrade

The cluster runs one Kubernetes minor version. To upgrade, run four sequential hops.
Each hop upgrades all 9 nodes: control plane first (serial: 1), workers in pairs (serial: 2).
The cluster stays available throughout — nodes are drained and uncordoned one at a time.

Current versions (as of 2026-05-17):

| Hop | From  | To      | Command |
|-----|-------|---------|---------| 
| 1   | 1.32  | 1.33.12 | `ansible-playbook playbooks/k8s-upgrade.yml -e "target_version=1.33.12 target_minor=1.33"` |
| 2   | 1.33  | 1.34.8  | `ansible-playbook playbooks/k8s-upgrade.yml -e "target_version=1.34.8  target_minor=1.34"` |
| 3   | 1.34  | 1.35.5  | `ansible-playbook playbooks/k8s-upgrade.yml -e "target_version=1.35.5  target_minor=1.35"` |
| 4   | 1.35  | 1.36.1  | `ansible-playbook playbooks/k8s-upgrade.yml -e "target_version=1.36.1  target_minor=1.36"` |

Run from `~/ansible-playbooks/` on ansible01, appending `--ask-vault-pass` to
each command (see [Vault password](#vault-password--required-for-every-playbook)).
Wait for each hop to complete before starting the next.

### What each hop does

1. Updates `/etc/apt/sources.list.d/kubernetes.list` to the new minor-version repo on every node
2. Upgrades kubeadm, runs `kubeadm upgrade apply` on k8scp01, `kubeadm upgrade node` on the rest
3. Re-applies CP static-pod customizations that kubeadm resets: rebinds etcd/controller-manager/scheduler
   metrics to `0.0.0.0`, and re-hardens the apiserver/etcd liveness probes + KCM/scheduler leader-election
   timeouts (see [control-plane hardening](#control-plane-hardening))
4. Drains each node, upgrades kubelet + kubectl, restarts kubelet, uncordons

### Before running

Check that all nodes are Ready and no PVCs are Pending:

```bash
kubectl get nodes
kubectl get pvc -A | grep -v Bound
```

### Checking available patch versions

If a newer patch is available when you run a hop:

```bash
curl -sL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Packages \
  | grep -A2 "^Package: kubeadm$" | grep Version | sort -V | tail -1
```

## OS patching (apt full-upgrade + reboot)

`os-patch.yml` is the **general, reusable** OS patcher — routine Ubuntu point
upgrades (e.g. 24.04.x → 24.04.4), kernel/security updates, any `apt
full-upgrade` that may need a reboot. It is **not** Kubernetes-version-specific
(it never touches kubeadm/kubelet/kubectl and never rebinds CP metrics — an apt
upgrade preserves the static-pod manifests).

```bash
# all control-plane nodes, one at a time
ansible-playbook playbooks/os-patch.yml -e patch_hosts=control_plane --ask-vault-pass
# a single node
ansible-playbook playbooks/os-patch.yml -e patch_hosts=k8scp01.vollminlab.com --ask-vault-pass
# all workers
ansible-playbook playbooks/os-patch.yml -e patch_hosts=workers --ask-vault-pass
```

`patch_hosts` defaults to `k8s` (every node). Always `serial: 1`. Per node it
drains, runs `apt full-upgrade` (with `force-confdef,force-confold` so a conffile
prompt never blocks the run), reboots only if `/var/run/reboot-required` is
present, waits for the node to report Ready, then uncordons.

The gate after each node depends on the node's role:

- **Control plane** — etcd-health gate **before** drain (all 3 members healthy)
  and an etcd-rejoin gate **after** reboot, because a reboot restarts the etcd
  static pod and 3-node etcd tolerates only one member down.
- **Workers** — a Longhorn gate: cleans up stale iSCSI sessions left by the
  reboot, then blocks until every volume is healthy and **hard-fails on any
  faulted volume** before moving to the next node. Volumes degraded only by
  `ReplicaSchedulingFailure` (no room for a 3rd replica) are treated as
  acceptable and don't block the roll.

Run only when the cluster is healthy (all etcd members healthy, all Longhorn
volumes healthy). The cluster stays available throughout.

## Control-plane hardening

`harden-cp-probes.yml` applies two independent shock absorbers to the static-pod
manifests so a transient etcd disk-latency blip can't cascade into a cluster-wide
control-plane restart storm.

**1. Liveness probes (kube-apiserver + etcd).** Raises each liveness
`failureThreshold` from `8` to `24`; with `periodSeconds=10` that widens tolerance
from ~80s to ~240s, so a blip no longer lets kubelet SIGKILL apiserver/etcd.

*Why:* on 2026-06-19 a brief storage I/O stall spiked etcd WAL fsync to ~1s. All
three apiservers' `/livez` failed, the kubelet SIGKILLed them, and every
`--leader-elect` controller in the cluster restarted (11 `PodCrashLooping` alerts).

**2. Leader-election timeouts (kube-controller-manager + kube-scheduler).** The
kubeadm defaults run these controllers with a bare `--leader-elect=true`, which
gives them only 10s to renew a lease (lease 15s / renew 10s / retry 2s). This play
adds explicit flags doubling every window — lease `30s`, renew-deadline `20s`,
retry-period `4s` — so a controller can ride out a multi-second etcd fsync spike
without losing leadership and restarting.

*Why:* on 2026-07-04 the nightly Velero `daily-full` backup (kopia FSB, one
PodVolumeBackup per volume) spiked etcd WAL fsync from a ~14ms baseline to 54ms.
That blew the 10s renew deadline and knocked four leader-elected controllers into
simultaneous restarts. Probe widening addresses the SIGKILL mode; leader-election
widening addresses the lease-loss mode.

Both are shock absorbers — they tolerate a blip but don't remove it. The trigger
itself (etcd sharing a physical ZFS pool with noisy neighbours) is removed by
storage isolation onto per-host local NVMe; see the `etcd-local-nvme-migration`
runbook in the k8s cluster repo.

```bash
ansible-playbook playbooks/harden-cp-probes.yml --ask-vault-pass
```

The play is **idempotent** and runs `serial: 1` with per-node readiness gates
(apiserver, etcd, controller-manager, scheduler), so etcd/apiserver quorum (2/3)
stays intact — only one CP node's components restart at a time. The same patches
are baked into `k8s-upgrade.yml` because kubeadm regenerates the static-pod
manifests (resetting the probes to `8` and dropping the leader-election flags) on
every upgrade.

## Docker Hub pull-through cache (containerd mirror)

`containerd-dockerhub-mirror.yml` configures every node's containerd to pull
`docker.io` images through the Harbor `dockerhub-proxy` project. This caches
Docker Hub images in Harbor after the first pull, eliminating the anonymous
rate-limit `ImagePullBackOff` storms that hit on mass reschedules (the cluster
shares one egress IP against Docker Hub's ~100 anonymous pulls/6h limit).

It is transparent — image refs stay `docker.io/...` and containerd silently
resolves them via Harbor. `registry-1.docker.io` remains the fallback host, so
if Harbor is down, pulls go directly to Docker Hub.

Prereq: the Harbor proxy-cache project must exist (created in the k8s repo) and
be public. Harbor uses a publicly-trusted Let's Encrypt cert, so no CA injection
is needed on the nodes.

```bash
ansible-playbook playbooks/containerd-dockerhub-mirror.yml --ask-vault-pass
```

What it does, per node (`serial: 1`):

1. Sets `config_path = "/etc/containerd/certs.d"` in the CRI registry block of
   `/etc/containerd/config.toml` (the identical line under the transfer plugin
   is deliberately left untouched).
2. Writes `/etc/containerd/certs.d/docker.io/hosts.toml` pointing at the proxy.
3. Restarts containerd (no drain — a containerd restart does not evict pods).
4. Verifies `crictl pull docker.io/library/busybox:latest` succeeds and the node
   returns Ready before moving to the next node.

Idempotent: re-running is a no-op once applied. New nodes should run this
playbook before joining production workloads.

## kubelet vm-lt mount ordering (k8sworker01)

`kubelet-vm-lt-mount-ordering.yml` installs a kubelet.service drop-in on
**k8sworker01 only** that adds `RequiresMountsFor=/mnt/vm-lt`, so kubelet
refuses to start until the VictoriaMetrics cold-tier filesystem is mounted.

Why: the cold tier (`victoria-metrics-lt`) stores ~13 months of metrics on a
750G ext4 filesystem at `/mnt/vm-lt`, surfaced as a static `local` PV. kubelet
bind-mounts `/mnt/vm-lt` into the pod — but if the filesystem is not mounted
when kubelet sets up the volume, kubelet silently binds the empty underlying
directory on the root LV instead, with no error. On 2026-07-08 that happened
(disk hot-added but never `mount -a`d, fstab carried `nofail`) and VM wrote
cold-tier data to the OS root disk until it was caught. The drop-in turns that
silent wrong-target bind into a hard systemd ordering dependency.

Scoped to k8sworker01 because no other node has `/mnt/vm-lt`; a cluster-wide
`RequiresMountsFor` would stop kubelet from starting elsewhere. The playbook
refuses to install unless `/mnt/vm-lt` is currently mounted **and** present in
`/etc/fstab`, and runs `daemon-reload` only (no kubelet restart) — the ordering
takes effect on the next kubelet start / reboot.

```bash
ansible-playbook playbooks/kubelet-vm-lt-mount-ordering.yml --ask-vault-pass
```

## Control-plane containerd upgrade

`cp-containerd-upgrade.yml` brings the 3 control-plane nodes from containerd
1.7.27 to 2.2.4 (the workers are already on 2.2.4). Roadmap item 7.2.

```bash
ansible-playbook playbooks/cp-containerd-upgrade.yml --ask-vault-pass
```

Why it's more careful than a worker upgrade despite being only 3 nodes:

- Each CP node runs an **etcd member** as a static pod. Upgrading the
  `containerd.io` package restarts containerd, which restarts every static pod
  on that node. 3-node etcd tolerates only one member down, so it runs
  `serial: 1` with an **etcd-health gate** before and after each node.
- `/etc/containerd/config.toml` is a dpkg conffile that is locally modified (it
  carries the Harbor pull-through mirror `config_path`). The upgrade runs with
  `--force-confold` to keep the existing file, then **re-asserts and verifies**
  the mirror config before restarting — so the docker.io proxy can't silently
  regress on these nodes.

Run only when all etcd members are healthy. The control plane stays available
throughout (nodes are drained and uncordoned one at a time).

## SSH key

`~/.ssh/ansible_k8s_ed25519` is a dedicated ed25519 key pair generated on ansible01.
Its public key is authorized in `~/.ssh/authorized_keys` on every managed node.
The private key lives only on ansible01 — it is not in 1Password and not forwarded via agent.

`~/.ssh/github_ansible_deploy` is the GitHub deploy key for this repo (push access only).
