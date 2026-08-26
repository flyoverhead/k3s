# `flyoverhead.k3s.resources`

Applies Kubernetes manifests to the cluster: from `resource_definition` dicts, template files, or YAML files. Supports create, update and delete operations.

## Role variables

| Variable | Description | Default |
| :--- | :--- | :--- |
| `resources_kubeconfig_local_path` | Path to kubeconfig on the controller | `~/.kube/config` |
| `resources_definition` | List of manifests to apply | `[]` |

Each entry in `resources_definition` supports these keys:

| Key | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | string | yes | Identifier for logging and loop control |
| `resource_definition` | dict/list | no | Inline manifest dict or list of dicts (YAML) |
| `src` | string | no | Path to YAML file containing the manifest |
| `template` | string | no | Path to Jinja template rendering a YAML manifest |
| `apply` | bool | no | Use `kubectl apply` instead of `kubectl create`; patches existing resources |
| `delete_all` | bool | no | Delete all resources matching the selector, not just the named one |
| `force` | bool | no | Force delete (applies grace period override, rarely needed) |
| `state` | string | no | Resource state: `present` or `absent` (default: `present`) |
| `wait` | bool | no | Block until the resource reports Ready (default: `true`) |

At least one of `resource_definition`, `src`, or `template` must be provided.

## Dependencies

The controller host must have:
- kubeconfig with cluster API access
- `jsonpatch` Python package (for strategic merge patches)

## Example playbook

```yaml
- hosts: localhost
  gather_facts: false
  vars:
    resources_definition:
      - name: test-namespace
        resource_definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: testing
      - name: from-file
        src: manifests/service.yaml
      - name: from-template
        template: templates/deployment.yaml.j2
  roles:
    - role: flyoverhead.k3s.resources
```

Delete a resource:

```yaml
resources_definition:
  - name: cleanup
    resource_definition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: old-config
        namespace: default
    state: absent
```

## Check mode

`--check --diff` reports what manifests would be created or deleted, and shows the computed YAML before it would be applied.
