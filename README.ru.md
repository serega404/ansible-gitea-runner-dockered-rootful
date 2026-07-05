# gitea_runner_docker

> Ansible-роль для установки Gitea Runner через Docker Compose в rootful-режиме.

[English version](README.md)

Роль устанавливает постоянный Gitea Actions runner в `/srv/gitea_runner` и
управляет им через Docker Compose.

## Что Делает Роль

- Ставит Docker только если он отсутствует.
- На не-Alpine Linux использует `curl -sSL https://get.docker.com | sh`.
- На Alpine ставит Docker системными пакетами.
- Создает файлы runner в `/srv/gitea_runner`.
- Хранит состояние runner в `/srv/gitea_runner/data`.
- Монтирует `/var/run/docker.sock` для rootful job-контейнеров.
- Рендерит конфиги `.env`, `config.yaml` и `compose.yaml`.
- Запускает `docker compose up -d`, когда меняются сгенерированные файлы.

## Требования

- Ansible 2.14 или новее.
- Registration token для Gitea runner.
- Linux-хост с поддержкой Docker.
- Root юзер на целевом хосте.

## Быстрый Старт

В репозитории уже есть готовый playbook для хоста из `inventory.yml`:

```bash
ansible-playbook playbooks/install.yml
```

Playbook спросит:

- имя runner и postfix для каталога в `/srv`, по умолчанию `main`;
- версию runner, по умолчанию `2.0`;
- URL Gitea;
- registration token.

Если оставить имя `main`, файлы будут созданы в `/srv/gitea_runner_main`.
Версия `2.0` автоматически превращается в Docker tag
`docker.io/gitea/runner:2.0.0`.

Роль также можно использовать из своего playbook:

```yaml
- name: Install Gitea Runner
  hosts: runners
  become: true
  roles:
    - role: gitea_runner_docker
      vars:
        gitea_runner_instance_url: https://git.example.com
        gitea_runner_registration_token: "{{ vault_gitea_runner_token }}"
```

Запуск:

```bash
ansible-playbook -i inventory.yml playbook.yml
```

## Основные Переменные

| Переменная                           | По умолчанию                   | Описание                                                                       |
| ------------------------------------ | ------------------------------ | ------------------------------------------------------------------------------ |
| `gitea_runner_dir`                   | `/srv/gitea_runner`            | Каталог для Compose-файлов и persistent data.                                  |
| `gitea_runner_image`                 | `docker.io/gitea/runner:2.0.0` | Образ runner.                                                                  |
| `gitea_runner_instance_url`          | `""`                           | URL Gitea, обязательная переменная.                                            |
| `gitea_runner_registration_token`    | `""`                           | Registration token, обязательная переменная. Задачи с ним используют `no_log`. |
| `gitea_runner_name`                  | `{{ inventory_hostname }}`     | Имя runner в Gitea.                                                            |
| `gitea_runner_labels`                | Ubuntu/Node defaults           | Labels, которые runner объявляет Gitea.                                        |
| `gitea_runner_install_docker`        | `true`                         | Ставить Docker, если `docker --version` не работает.                           |
| `gitea_runner_manage_docker_service` | `true`                         | Включать и запускать Docker service.                                           |
| `gitea_runner_manage_compose`        | `true`                         | Запускать `docker compose up -d` через handler.                                |
| `gitea_runner_docker_socket`         | `/var/run/docker.sock`         | Путь к Docker socket на хосте.                                                 |
| `gitea_runner_compose_project_name`  | `gitea_runner`                 | Название compose проекта.                                                      |
| `gitea_runner_restart_policy`        | `unless-stopped`               | Политика перезапуска контейнера.                                               |
| `gitea_runner_cache_enabled`         | `false`                        | Включить настройки cache.                                                      |
| `gitea_runner_cache_host`            | `""`                           | Host/IP, доступный job-контейнерам при включенном cache.                       |
| `gitea_runner_cache_port`            | `8088`                         | Cache порт.                                                                    |
| `gitea_runner_cache_dir`             | `""`                           | Cache папка внутри контейнера.                                                 |
| `gitea_runner_config`                | `{}`                           | Переопределение для генерируемого `config.yaml`.                               |
| `gitea_runner_config_raw`            | `""`                           | Полный raw `config.yaml`; если задан, шаблон не генерируется.                  |
| `gitea_runner_environment_extra`     | `{}`                           | Дополнительные переменные окружения, записываемые в `.env`.                    |

Готовый `playbooks/install.yml` вычисляет эти переменные из ответов:

- `gitea_runner_dir`: `/srv/gitea_runner_<normalized_name>`;
- `gitea_runner_name`: введенное имя runner;
- `gitea_runner_image`: `docker.io/gitea/runner:<version>`, при этом `2.0`
  нормализуется в `2.0.0`;
- Compose project и container name: `gitea_runner_<normalized_name>`.

## Cache

! Не тестировалось.

Cache выключен по умолчанию. Для включения укажите адрес хоста, доступный из
job-контейнеров:

```yaml
gitea_runner_cache_enabled: true
gitea_runner_cache_host: 192.168.1.10
gitea_runner_cache_port: 8088

## Лицензия

[MIT](./LICENSE)
