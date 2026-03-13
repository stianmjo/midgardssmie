# verdenssmie

> *"World Forge"* — forges bare metal into a living Kubernetes cluster.

Ansible playbook that takes a fresh Ubuntu node from zero to a fully running
Kubernetes cluster with Cilium CNI and Flux GitOps. Once Flux is up it
reconciles your homelab repo and the entire stack comes up on its own.

---

## What it does

| Phase | Role | What happens |
|-------|------|--------------|
| 1 | `common` | Packages, SSH keys, swap off, kernel modules, sysctl, multipathd, containerd |
| 2 | `kubeadm` | Kubernetes APT repo, kubeadm init, kubeconfig fetched locally |
| 3 | `local-path` | Rancher local-path-provisioner, set as default StorageClass |
| 4 | `cilium` | Cilium CNI + kube-proxy replacement + Hubble |
| 5 | `flux` | Flux operator, github-credentials secret, FluxInstance applied |

After phase 5, Flux reconciles `cluster/` from your homelab repo and
everything else is GitOps.

---

## Prerequisites

On the machine running the playbook:

```bash
pip install ansible kubernetes
ansible-galaxy collection install kubernetes.core community.general ansible.posix
```

---

## Setup

### 1. Clone

```bash
git clone <your-repo-url>
cd verdenssmie
```

### 2. Fill in your environment file

```bash
cp group_vars/all/env.template.yaml group_vars/all/env.yaml
```

Edit `env.yaml` with your values — this file is gitignored and never committed:

```yaml
flux_github_token: "github_pat_..."
ssh_authorized_keys:
  - "ssh-ed25519 ..."
```

### 3. Fill in your inventory

Edit `inventory/hosts.yaml` with your node IPs and usernames.

### 4. Run

```bash
ansible-playbook site.yaml
```

Or run individual phases:

```bash
ansible-playbook site.yaml --tags common       # host prep only
ansible-playbook site.yaml --tags kubeadm      # kubeadm only
ansible-playbook site.yaml --tags local-path   # local-path-provisioner only
ansible-playbook site.yaml --tags cilium       # Cilium only
ansible-playbook site.yaml --tags flux         # Flux only
```

---

## After the playbook

A kubeconfig is fetched to `./kubeconfig` in the repo root (gitignored).
Use it to interact with the cluster:

```bash
export KUBECONFIG=./kubeconfig
kubectl get nodes
```

App secrets must be applied manually before workloads come up.
Check each `*-secret-template.yaml` in your homelab repo and apply them.

---

## Adding worker nodes

1. Uncomment the `workers` group in `inventory/hosts.yaml` and add IPs
2. Uncomment the worker join tasks in `roles/kubeadm/tasks/main.yaml`
3. Run: `ansible-playbook site.yaml --tags kubeadm --limit workers`

---

## Stack versions

| Component | Version |
|-----------|---------|
| Kubernetes | 1.35 |
| Cilium | 1.19.1 |
| Flux operator | 0.43.0 |
| Flux | 2.x |
| local-path-provisioner | 0.0.32 |

Versions are defined in `group_vars/all/main.yaml` and can be updated there.