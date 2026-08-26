# `flyoverhead.k3s.helm`

Manages Helm repositories and chart releases: adds or updates repository endpoints, installs or upgrades releases with custom values from templates, files or inline YAML.

## Role variables

| Variable | Description | Default |
| :--- | :--- | :--- |
| `helm_kubeconfig_local_path` | Path to kubeconfig on the controller | `~/.kube/config` |
| `helm_charts` | List of chart releases to install or upgrade | `[]` |

Each entry in `helm_charts` supports these keys:

| Key | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | string | yes | Identifier for logging and loop control |
| `repo_name` | string | yes | Helm repository name (used to construct repository identity) |
| `repo_url` | string | yes | Helm repository URL |
| `chart_ref` | string | yes | Chart reference: `<repo_name>/<chart_name>` |
| `chart_version` | string | yes | Chart version (SemVer) |
| `release_name` | string | yes | Helm release name (used by `helm list`) |
| `release_namespace` | string | no | Target namespace (default: `kube-system`) |
| `release_values` | dict | no | Values dict to override the chart's defaults |
| `create_namespace` | bool | no | Create the target namespace if missing (default: `true`) |
| `update_repo_cache` | bool | no | Fetch fresh repository index (default: `true`) |
| `reset_values` | bool | no | Discard the chart's default values (default: `false`) |
| `reuse_values` | bool | no | Merge with previous release's values (default: `false`, see note below) |
| `set_values` | dict | no | Values to set via `--set` flag |
| `state` | string | no | Release state: `present` or `absent` (default: `present`) |
| `wait` | bool | no | Block until release is healthy (default: `true`) |

**`reuse_values` defaults to `false` as of 2.0.0.** Cilium's upgrade guide states:

> When upgrading from one minor release to another minor release using `helm upgrade`, do *not* use Helm's `--reuse-values` flag. The `--reuse-values` flag ignores any newly introduced values present in the new release and thus may cause the Helm template to render incorrectly.

This applies beyond Cilium: this role exists to apply *declarative* values from templates and files, not to merge with whatever the cluster currently runs. Any chart whose live values were changed outside Ansible will show drift on the first apply.

## Dependencies

The controller host must have:
- kubeconfig with cluster API access
- `helm` binary on `$PATH`

## Example playbook

```yaml
- hosts: localhost
  gather_facts: false
  vars:
    helm_charts:
      - name: cilium
        repo_name: cilium
        repo_url: https://helm.cilium.io
        chart_ref: cilium/cilium
        chart_version: 1.14.0
        release_name: cilium
        release_namespace: kube-system
        release_values:
          ipam:
            mode: kubernetes
          hubble:
            ui:
              enabled: true
        wait: true
  roles:
    - role: flyoverhead.k3s.helm
```

Remove a release:

```yaml
helm_charts:
  - name: old-app
    chart_ref: old-app/old-app
    chart_version: 1.0.0
    release_name: old-app
    release_namespace: default
    state: absent
```

## Check mode

`--check --diff` reports what releases would be added or removed, and shows the computed release values before they would be applied.
