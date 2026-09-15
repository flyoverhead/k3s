# Changelog

All notable changes to `flyoverhead.k3s`.

## 3.0.0

Breaking: the bundled playbooks moved from `playbooks/tasks/` to
`playbooks/plays/`. The documented entry point, `playbooks/playbook.yml`, is
unchanged -- only a playbook that imported a sub-play by path needs updating.

### Changed

- **`playbooks/tasks/` -> `playbooks/plays/`.** The directory held plays, not task files, and ansible-lint infers file kind from the path, so the reserved `tasks/` name made it parse each play as a task list: `schema[tasks]: 'block' is a required property` and `parser-error: conflicting action statements: hosts, roles`. Import paths in `playbooks/playbook.yml` were updated to match.
- **`hosts:` now uses native host patterns.** `'{{ groups[k3s_server_group][0] }}'` became `'{{ k3s_server_group }}[0]'`, which no longer needs the inventory resolved at parse time. Host selection is unchanged, verified against a 3-server/2-agent inventory: `k3s_server` -> all three, `k3s_server[0]` -> the first, `k3s_server[1:]` -> the rest, `k3s_agent` -> both agents. `ha.yml` and `agent.yml` keep `order: inventory`.
- **`config.yml` and `uninstall.yml` default their group vars** to the same values `roles/k3s/defaults/main.yml` already declares, so each play is lintable standalone.

### Fixed

- **ansible-lint passes the production profile with 0 failures**, down from 10. Commits to this repository no longer need `--no-verify` to get past the pre-commit chain.
- Trailing whitespace in `playbooks/templates/external-dns/values.j2`.

### Known issues

- **A mistyped group name now skips a play instead of failing it.** `groups[missing_group]` raised a hard error; a host pattern that matches nothing warns and skips. `ha`, `single` and `agent` are still caught by the `when: groups[...] | length > 0` guards in `playbooks/playbook.yml`, but `config.yml` and `uninstall.yml` have no such guard.

## 2.2.0

### Changed

- **reflector v7.1.288 → 10.0.65.** Dependency updates only, no value or template changes required.
- **registry v2.8.3 → 3.1.1.** Removed support for oss/swift storage drivers and legacy libtrust; uses filesystem storage so no value changes required.
- **gitea chart v10.6.0 → v12.7.0.** Redis migrated to Valkey (`redis` → `valkey`, `redis-cluster` → `valkey-cluster`). Breaking: chart v12 outsourced the Actions sub-chart to `gitea/helm-actions`; the `actions:` block has been removed from this collection's values template. Deleted `gitea_runner_version` pin (now unused).
- **Deleted `registry_helm_version` pin.** Never read by the collection; dead variable removed from inventory.
- **cert-manager v1.17.2 → v1.21.1.** All schema keys present; `config.enableGatewayAPI` removed from chart but left in values (harmless, Helm ignores unknown keys).
- **pihole v2.27.0 → 2.38.0.** All schema keys present, number-only bump.
- **external-dns 1.16.1 → 1.21.1.** All schema keys present, number-only bump.
- **csi-driver-smb v1.18.0 → 1.20.3.** All schema keys present, number-only bump.
- **registry-ui image 2.5.7 → 2.6.0.** Number-only bump.
- **registry-ui chart 1.1.3 → 1.1.4.** All schema keys present, number-only bump.
- **minio v2024-12-18 → v2025-10-15.** All schema keys present, number-only bump.
- **minio-mc v2024-11-21 → v2025-08-13.** Number-only bump.
- **minio chart 5.3.0 → 5.4.0.** All schema keys present, number-only bump.

### Known issues

- **Gitea Actions runners no longer deployed.** Gitea chart v12 moved the Actions sub-chart to `gitea/helm-actions`, which this collection does not deploy. The `actions:` block has been removed from the collection's gitea values template. Anyone requiring CI runners must deploy `gitea/helm-actions` as a separate chart.
- **Not harness-covered.** Thirteen of the fourteen bumps (all except reflector) deploy with `k3s_apps_enabled: false` in the test suite and are not exercised. The harness covers core cluster bootstrap (k3s, gateway-api, cilium, reflector, kube-vip) only.

## 2.1.0

### Changed

- **k3s v1.33.0+k3s1 → v1.36.3+k3s1.** A skip of two Kubernetes minors; safely
  applied to a cluster rebuilt rather than upgraded in place, which both k3s and
  cilium forbid for skipped minors.
- **cilium v1.17.4 → 1.20.1.** Note: Helm publishes cilium chart versions
  without the leading `v`. cilium 1.20.1 is e2e-tested against Kubernetes
  1.33–1.36; k3s v1.36.3+k3s1 is Kubernetes 1.36, so the pair is within the
  support matrix.
- **gateway-api v1.2.1 → v1.6.1.**
- **kube-vip v0.9.1 → v1.2.3.** kube-vip 1.x removed the `vip_cidr`
  environment variable; `pkg/kubevip/config_envvar.go` and
  `pkg/vip/address.go` at v1.2.3 show `vip_subnet` in its place, and
  `SelectSubnet` takes a bare prefix length (comma-separated for dual-stack), so
  the value stays `"32"`, not `"/32"`. Also deleted the duplicate
  `vip_leaderelection` env entry that was listed twice.

### Known issues

- The harness runs one server, so HA and kube-vip failover are not exercised —
  kube-vip is not covered by the test suite.

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
