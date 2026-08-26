# `flyoverhead.k3s.k3s`

Bootstraps a k3s node from scratch, installs or updates the k3s binary and systemd service, generates kubeconfig, and on multi-server clusters applies kube-vip manifests for API load balancing before agents join. Node updates cordon and drain pods, wait for Ready, then uncordon. Updates deliberately leave a failed node cordoned for manual recovery.

## Role variables

| Variable | Description | Example |
| :--- | :--- | :--- |
| `k3s_version` | k3s release tag from GitHub, including patch and k3s build number | `v1.31.3+k3s1` |
| `k3s_server_group` | Name of the inventory group containing server nodes | `k3s_server` |
| `k3s_agent_group` | Name of the inventory group containing agent nodes | `k3s_agent` |
| `k3s_config.iface` | Network interface hosting the node IP (for kube-vip and agents' API endpoint detection) | `eth0` |
| `k3s_config.api_port` | k3s API server port | `6443` |
| `k3s_config.cluster_cidr` | Pod CIDR block allocated to the cluster, immutable on a live cluster | `10.52.0.0/16` |
| `k3s_config.lb_ip_range` | IP address range for kube-vip's load-balanced services (CIDR or dash-separated) | `192.168.1.150-192.168.1.160` |
| `k3s_kube_vip_config.version` | kube-vip release tag from GitHub | `v0.8.7` |
| `k3s_kube_vip_config.api_address` | Virtual IP address for the cluster API (answers on multi-server, given to agents) | `192.168.1.250` |
| `k3s_kube_vip_config.iface` | Network interface for kube-vip to bind to; inherits from `k3s_config.iface` | `eth0` |
| `k3s_kube_vip_config.lb_ip_range` | Service LB range; inherits from `k3s_config.lb_ip_range` | `192.168.1.150-192.168.1.160` |
| `k3s_kube_vip_resources` | Manifest definitions for kube-vip RBAC and DaemonSet (applied on the first server before agents join) | See defaults |
| `k3s_controller_config.address` | Host where `kubectl` commands run (usually localhost) | `localhost` |
| `k3s_controller_config.path` | Directory containing kubeconfig on the controller | `~/.kube` |
| `k3s_controller_config.name` | Kubeconfig filename | `config` |
| `k3s_controller_config.owner` | Owner of kubeconfig (user running cluster operations) | `aletunov` |
| `k3s_controller_config.group` | Group of kubeconfig on macOS | `staff` |
| `k3s_drain_enabled` | Cordon, drain and uncordon on node update | `true` |
| `k3s_drain_timeout` | Timeout in seconds for `kubectl drain` to complete | `300` |
| `k3s_uninstall` | Uninstall k3s, unmount directories and reboot (this is destructive) | `false` |

## Facts set by this role

| Fact | Description |
| :--- | :--- |
| `k3s_arch` | Architecture detected from `ansible_facts.architecture`: `amd64` or `arm64` |
| `k3s_binary` | k3s binary name: `k3s` (x86_64) or `k3s-arm64` (aarch64) |
| `k3s_installed` | Boolean: whether k3s.service exists in systemd |
| `k3s_install_version` | Version to be installed (extracted from `k3s_version`) |
| `k3s_installed_version` | Version currently installed (extracted from `k3s --version`) |
| `k3s_update` | Boolean: whether the install version is newer than the installed version |
| `k3s_node_ip` | Node's IP address on `k3s_config.iface` |
| `k3s_api_endpoint` | Endpoint that agents will use: VIP on multi-server, first server's IP on single-server |
| `k3s_cluster_configured` | Boolean: whether the cluster API is reachable from the controller |
| `k3s_node_joined` | Boolean: whether this node is already a cluster member |
| `k3s_token` | Node join token, read from `/var/lib/rancher/k3s/server/token` on the first server |
| `k3s_cluster_size` | Count of all servers and agents in the cluster |

## Dependencies

| Name | Description |
| :--- | :--- |
| `flyoverhead.k3s.resources` | Applies kube-vip manifests; the VIP must answer before agents are handed it as their API endpoint |

## Behaviour worth knowing before the first run

- On a single-node cluster, `k3s_drain_enabled: true` is silently skipped — draining the only node evicts every pod with nowhere to reschedule it.

- A failed update deliberately leaves the node cordoned. Recover it with `kubectl uncordon <node>` after you fix the issue.

- `k3s_uninstall: true` reboots the node unconditionally at the end, with no warning.

- `k3s_config.cluster_cidr` changes require a full rebuild: etcd holds already-allocated pod addresses. You cannot change it on a live cluster.

- The role reads `k3s_kube_vip_resources` to decide what manifests to apply to multi-server clusters; this variable previously lived in `playbooks/vars/main.yml` and was not documented here, making the role unusable standalone.

## Check mode

`--check --diff` works against an already-bootstrapped host: it reports which tasks would change the kubeconfig, config.yaml, and k3s.service. The three cluster-facing tasks — drain, Ready-wait and uncordon — report rather than act.

On a freshly provisioned host with no k3s yet, `--check` simulates the install and reports what it would do.

## Example playbook

```yaml
- hosts: k3s_server
  gather_facts: true
  roles:
    - role: flyoverhead.k3s.k3s

- hosts: k3s_agent
  gather_facts: true
  serial: '{{ k3s_agent_serial | default(1) }}'
  roles:
    - role: flyoverhead.k3s.k3s
```

Update a live cluster:

```yaml
- hosts: all
  gather_facts: true
  vars:
    k3s_version: v1.32.0+k3s1
  roles:
    - role: flyoverhead.k3s.k3s
```

Uninstall (destructive):

```yaml
- hosts: all
  gather_facts: true
  vars:
    k3s_uninstall: true
  roles:
    - role: flyoverhead.k3s.k3s
```
