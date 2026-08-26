# Changelog

All notable changes to `flyoverhead.k3s`.

## 2.0.0

### Fixed

- **Collection name.** `galaxy.yml` declared `name: k8s` while the directory,
  the symlink `new_lab` resolves through, and the repository all said `k3s`.
  A built artifact installed to `flyoverhead/k8s` and broke every FQCN.
- **Undeclared dependencies.** Only `kubernetes.core` was declared, but
  `roles/k3s` calls `ansible.posix.sysctl`, `ansible.posix.mount` and
  `community.general.alternatives`, and the playbook uses
  `ansible.utils.ipmath`. A clean install could not run the collection.
- **Check mode.** Nothing had ever set `check_mode`.
  `detect | get k3s installed version` is `ansible.builtin.command`, skipped
  in a check run, and the following `set_fact` dereferenced its `stdout_lines`
  — so `--check` aborted against any host that already had k3s.
- **Roles were not self-contained.** No role owned its own assets: all four
  roles' `files/` and `templates/` were symlinks to the same two shared
  `playbooks/` directories. The genuine playbook coupling was that `roles/k3s`
  read `kube_vip_resources` from `playbooks/vars/main.yml`. `flyoverhead.k3s.k3s`
  only worked when driven by the bundled playbook.
- **Duplicate kube-vip apply.** `playbooks/tasks/config.yml` applied
  `kube_vip_resources` unconditionally, including on single-server clusters,
  duplicating what `roles/k3s` already does conditionally.
- **Test harness could not complete.** Four variables the playbook reads were
  defined in no test var file, the LB pool and VIP were on subnets the
  `Vagrantfile` does not create, and `external_dns_version` was an app version
  in a slot consumed as `chart_version`.
- **`k3s_timezone`** was read by `pihole/values.j2` and defaulted nowhere.
- Removed `jmespath`, `passlib` and `pyhelm` from `requirements.txt`. None was
  referenced anywhere; `pyhelm` is unmaintained upstream.
- Deleted `playbooks/templates/kube-vip/configmap.j2`, referenced by nothing.

### Added

- **Node updates cordon, drain, wait for `Ready` and uncordon.** k3s's own
  upgrade documentation states the install script does none of this and the
  caller must. `ignore_daemonsets` is required rather than optional — cilium,
  kube-vip and csi-smb are DaemonSets and the drain cannot otherwise complete.
  Skipped on single-node clusters. `k3s_drain_enabled`, `k3s_drain_timeout`.
  A failed update deliberately leaves the node cordoned.
- **`k3s_agent_serial`**, default 1. Agents previously all stopped k3s at the
  same moment; k3s's docs say to upgrade servers one at a time, then agents.
- **`k3s_apps_enabled`**, default true. The harness sets it false to scope
  itself to the core cluster.
- READMEs for the collection and all four roles. `roles/k3s` had none, and the
  other three referenced a `flyoverheadk8s` namespace that does not exist —
  `roles/patch`'s was a copy of `roles/resource`'s, including its title.

### Changed

- **BREAKING: `patch` → `patches`, `resource` → `resources`.** Variables
  (`patches_definition`, `resources_definition`, and their
  `*_kubeconfig_local_path` companions) are unchanged.
- **BREAKING: `reuse_values` now defaults to `false`.** Cilium's upgrade guide
  says not to use `helm upgrade --reuse-values` across minors because it
  ignores newly introduced values; it is wrong generally for a role that
  applies declarative values from templates. Charts whose live values were
  changed outside Ansible will show drift on the first apply.
- Role `meta/main.yml` ×4: `license: GNU` was not an SPDX identifier,
  `min_ansible_version` contradicted `meta/runtime.yml`, `role_name` and
  `namespace` were absent, and Trixie was missing from `platforms`.
- `archive_download` → `k3s_archive_download`.

### Known issues

- The harness runs one server, so HA and kube-vip failover are not covered —
  including the `vip_subnet` rename shipped in 2.1.0.
- `roles/k3s/tasks/uninstall.yml` reboots the node unconditionally.

## 1.0.0

Initial collection.
