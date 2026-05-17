# ansible-playbooks

Ansible automation for the Vollminlab homelab. Runs from `ansible01.vollminlab.com`.

## Layout

```
inventory/
  hosts.ini          # all managed hosts, grouped by role
playbooks/
  k8s-upgrade.yml    # Kubernetes minor-version upgrade, one hop at a time
ansible.cfg          # default inventory, SSH key, connection settings
```

## Prerequisites

ansible01 must have:
- Ansible (`ansible --version`)
- kubectl + `~/.kube/config` pointing at the cluster
- `~/.ssh/ansible_k8s_ed25519` — the dedicated Ansible SSH key (authorized on all nodes)

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

Run from `~/ansible-playbooks/` on ansible01. Wait for each hop to complete before starting the next.

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

## SSH key

`~/.ssh/ansible_k8s_ed25519` is a dedicated ed25519 key pair generated on ansible01.
Its public key is authorized in `~/.ssh/authorized_keys` on every managed node.
The private key lives only on ansible01 — it is not in 1Password and not forwarded via agent.

`~/.ssh/github_ansible_deploy` is the GitHub deploy key for this repo (push access only).
