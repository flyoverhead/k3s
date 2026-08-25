# `flyoverhead.k3s.helm`

Роль для установка и обновления helm чартов

Поддерживаемый ролью функционал:
- установка, обновление и удаление чартов

[[_TOC_]]

## Переменные роли

| Имя | Пример | Описание |
| :--- | :--- | :--- |
| **helm_kubeconfig_local_path** | `'{{ ansible_user_dir }}/.kube/config'` | Путь к каталогу с файлом конфигурации кластера |
| **helm_charts** | Пример в [defaults](./defaults/main.yml) | Список helm чартов, которые необходимо установить/обновить |

## Примеры настройки

### Конфигурация

```yaml
---

helm_charts:
  - name: cilium
    repo_url: https://helm.cilium.io
    chart_ref: cilium/cilium
    chart_version: '{{ kubernetes_cilium_version }}'
    release_namespace: kube-system
    release_values: >-
      {{ lookup('template',
      kubernetes_data_path + '/cni/cilium/' + kubernetes_cilium_release |
      trim + '/config.yml.j2') | from_yaml }}
```

> В качестве значения переменной `release_values` списка словарей `helm_charts` может быть указан любой YAML файл конфигурации или jinja шаблон, с указанием полного пути.

### Плейбук

```yaml
---

- name: helm
  hosts: k8s

  roles:
    - role: flyoverheadk8s.helm
```
