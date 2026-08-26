# `flyoverhead.k3s.patches`

Applies JSON patches (RFC 6902) against existing cluster resources: add, remove or replace fields in Deployments, ConfigMaps, Services, and other Kubernetes objects.

## Role variables

| Variable | Description | Default |
| :--- | :--- | :--- |
| `patches_kubeconfig_local_path` | Path to kubeconfig on the controller | `~/.kube/config` |
| `patches_definition` | List of patches to apply | `[]` |

Each entry in `patches_definition` supports these keys:

| Key | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | string | yes | Resource name (used by `kubectl`) |
| `kind` | string | yes | Resource kind (Deployment, ConfigMap, Service, etc.) |
| `namespace` | string | no | Target namespace (default: `kube-system`) |
| `patch` | list | yes | JSON patch operations: array of `{op, path, value}` dicts |
| `wait` | bool | no | Block until the patched resource reports Ready (default: `true`) |

## Dependencies

The controller host must have:
- kubeconfig with cluster API access
- `jsonpatch` Python package

## Example playbook

```yaml
- hosts: localhost
  gather_facts: false
  vars:
    patches_definition:
      - name: coredns
        kind: Deployment
        namespace: kube-system
        patch:
          - op: add
            path: /spec/template/spec/affinity
            value:
              podAntiAffinity:
                preferredDuringSchedulingIgnoredDuringExecution:
                  - weight: 100
                    podAffinityTerm:
                      labelSelector:
                        matchExpressions:
                          - key: k8s-app
                            operator: In
                            values:
                              - kube-dns
                      topologyKey: kubernetes.io/hostname
  roles:
    - role: flyoverhead.k3s.patches
```

## Check mode

`--check --diff` reports what patches would be applied and shows the patch operations before they would be executed.
