# `flyoverhead.k3s`

[![Version](https://img.shields.io/badge/version-2.2.0-blue)](galaxy.yml)
[![ansible-core](https://img.shields.io/badge/ansible--core-%E2%89%A52.16-black?logo=ansible&logoColor=white)](https://docs.ansible.com/ansible-core/devel/index.html)
[![License](https://img.shields.io/badge/license-GPL--3.0--only-green)](https://www.gnu.org/licenses/gpl-3.0)
[![Platform](https://img.shields.io/badge/platform-Debian%2012%20%7C%2013-A81D33?logo=debian&logoColor=white)](#-supported-os)
[![Roles](https://img.shields.io/badge/roles-4-orange)](#-roles)

K3s cluster deployment and management: bootstrapping nodes with k3s binaries and kubeconfig, joining agents, updating nodes with cordon/drain/uncordon, and coordinating Helm releases, Kubernetes manifests, and JSON patches across the cluster.

## 🚀 Quick Start

### Requirements

- `ansible-core >=2.16`

- Collections: `ansible.posix >=1.5.4`, `ansible.utils >=2.5.0`, `community.general >=8.0.0`, `kubernetes.core >=3.4.3`

- `kubernetes` and `jsonpatch` on the controller

- **`helm` binary on the controller** — `kubernetes.core.helm` shells out to it, and nothing in this collection installs it

- Task `>=3.20`, Vagrant and a VMware Fusion provider, for the test harness only

### Installation

Installing the collection dependencies:

```bash
ansible-galaxy collection install -r requirements.yml
pip install -r requirements.txt
```

Installing the collection itself:

```bash
ansible-galaxy collection install git+https://github.com/flyoverhead/k3s
```

### Roles usage

Full documentation and usage examples of role `<role>` can be found in `roles/<role>/README.md`.

Run `flyoverhead.k3s.k3s` on both server and agent nodes. It detects whether k3s is installed, installs it if not, updates it if a newer version is declared, and applies the k3s configuration. On multi-server clusters, it applies kube-vip for HA before agents join, because agents are given the VIP as their API endpoint.

Use `flyoverhead.k3s.helm`, `flyoverhead.k3s.resources` and `flyoverhead.k3s.patches` to manage cluster state after bootstrap. They depend on the controller host having a kubeconfig and access to the cluster API.

### Example Playbook

```yaml
---
- name: k3s cluster
  hosts: all
  gather_facts: true
  serial: 1  # servers first, sequentially
  roles:
    - role: flyoverhead.k3s.k3s

- name: k3s agents
  hosts: '{{ groups["k3s_agent"] }}'
  gather_facts: true
  serial: '{{ k3s_agent_serial | default(1) }}'  # increase for faster first build
  roles:
    - role: flyoverhead.k3s.k3s

- name: cluster apps
  hosts: localhost
  gather_facts: false
  roles:
    - role: flyoverhead.k3s.helm
    - role: flyoverhead.k3s.resources
    - role: flyoverhead.k3s.patches
```

## Collection variables

These variables control cluster-wide behavior. They live in `playbooks/vars/main.yml` and can be overridden per-play.

| Variable | Default | Description |
| :--- | :--- | :--- |
| `k3s_agent_serial` | `1` | Agents per task run (raise to parallelise a first build; serial is play-level and cannot be conditional on update vs. install) |
| `k3s_apps_enabled` | `true` | Deploy the optional app stack (pihole, minio, gitea, registry, csi-smb, external-dns); the test harness sets false to scope to the core cluster |
| `k3s_timezone` | `Etc/UTC` | Timezone used by pihole and other apps that consume it; set this before deploy if the cluster runs in a different region |

## 🖥 Supported OS

| OS | Status |
| :--- | :--- |
| Debian 12 "Bookworm" (AArch64, x86_64) | Tested |
| Debian 13 "Trixie" | Supported, untested |

Everything is apt-based and Debian-family only. k3s provides binaries for both architectures and both distributions.

## 📦 Roles

| Name | Description |
| :--- | :--- |
| [`k3s`](roles/k3s/README.md) | Cluster node: binary, config.yaml, k3s.service, kubeconfig, kube-vip for HA, safe updates |
| [`helm`](roles/helm/README.md) | Helm repositories and chart releases |
| [`resources`](roles/resources/README.md) | Kubernetes manifests, from templates, files or inline definitions |
| [`patches`](roles/patches/README.md) | JSON patches against existing cluster resources |

## ⚠️ Gotchas

Four things here deserve your attention before a first deploy:

- **`roles/k3s` uninstall reboots the node, unconditionally.** `tasks/uninstall.yml` ends in `ansible.builtin.reboot` with no guard. Set `k3s_uninstall: true` only when you intend a destructive teardown.

- **`k3s_config.cluster_cidr` cannot be changed on a live cluster.** etcd holds already-allocated pod addresses; changing it requires a full cluster rebuild with new data directories.

- **Agent updates are serial by default, including on a fresh install.** `serial` is play-level and evaluated before any facts exist, so it cannot be conditional on whether the run is an upgrade. Raise `k3s_agent_serial` to parallelise a first build if speed matters.

- **`reuse_values` defaults to `false` as of 2.0.0.** Every chart re-renders from its `values.j2` on each run. Any chart whose live values were changed outside Ansible will show drift on the first apply.

## 🧪 Testing

```bash
task deploy      # vagrant up, then run tests/playbook.yml against it
task provision   # re-run the playbook
task destroy
```

`task deploy` covers k3s install, agent join and update, plus gateway-api, cilium and reflector. It does **not** cover HA or kube-vip failover — the harness runs one server — and it does not cover csi-smb, minio, registry, gitea, pihole or external-dns, which `k3s_apps_enabled: false` disables because they need an SMB share and vaulted credentials.

## 📄 License

GPL-3.0-only.
