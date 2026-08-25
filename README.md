# `flyoverhead.k3s`

K3S cluster deployment and management

Поддерживаемый коллекцией функционал:
- развёртывание кластера
- установка и обновление helm чартов
- применение патчей
- установка и обновление ресурсов

[[_TOC_]]

## Поддерживаемые ОС

- Debian 12.x "Bookworm"

## Использование

```yaml
---
- name: deploy k3s
  ansible.builtin.import_playbook: flyoverhead.k3s.playbook
```

### Роли

| Имя | Описание |
| :--- | :--- |
| [kubernetes](roles/kubernetes/README.md) | Развёртывание кластера |
| [helm](roles/helm/README.md) | Установка и обновление helm чартов |
| [patches](roles/patches/README.md) | Установка патчей |
| [resources](roles/resources/README.md) | Установка и обновление ресурсов |
