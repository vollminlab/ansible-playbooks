# ansible-playbooks

Ansible automation for the Vollminlab homelab. Runs from `ansible01.vollminlab.com`.

## Layout

```
inventory/
  hosts.ini                       # all managed hosts, grouped by role
playbooks/
  k8s-upgrade.yml                 # Kubernetes minor-version upgrade, one hop at a time
  containerd-dockerhub-mirror.yml # route docker.io through the Harbor pull-through cache
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
3. Drains each node, upgrades kubelet + kubectl, restarts kubelet, uncordons

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
