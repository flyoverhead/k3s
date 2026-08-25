# flyoverheadk8s.resources

Роль для установки и обновления ресурсов кластера k8s

Поддерживаемый ролью функционал:
- установка, обновление и удаление ресурсов

[[_TOC_]]

## Переменные роли

| Имя | Пример | Описание |
| :--- | :--- | :--- |
| **resources_kubeconfig_local_path** | `'{{ ansible_user_dir }}/.kube/config'` | Путь к каталогу с файлом конфигурации кластера |
| **resources_data_path** | `{{ inventory_dir }}/data/kubernetes` | Путь к каталогу с файлами конфигураций (values) для чартов |
| **resources_definition** | Пример в [defaults](./defaults/main.yml) | Список ресурсов, которые необходимо установить/обновить/удалить |

## Примеры настройки

### Конфигурация

```yaml
---

resources_kubeconfig_local_path: '{{ ansible_user_dir }}/.kube/config'
resources_data_path: '{{ inventory_dir }}/data/kubernetes'

resources_definition:
  - name: ceph
    template:
      - path: '{{ kubernetes_data_path + "/csi/ceph/secret.yml.j2" }}'
      - path: '{{ kubernetes_data_path + "/csi/ceph/rbac.yml.j2" }}'
      - path: '{{ kubernetes_data_path + "/csi/ceph/provisioner.yml.j2" }}'
      - path: '{{ kubernetes_data_path + "/csi/ceph/plugin.yml.j2" }}'
```

### Плейбук

```yaml
---

- name: resources
  hosts: k8s

  roles:
    - role: flyoverheadk8s.resources
```
