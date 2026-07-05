# gitea_runner_docker

> Ansible role for installing Gitea Runner with Docker Compose in rootful mode.

Language: [Русский](README.ru.md)

This role installs a persistent Gitea Actions runner into `/srv/gitea_runner`
and manages it with Docker Compose.

## What The Role Does

- Installs Docker only when it is missing.
- Uses `curl -sSL https://get.docker.com | sh` on non-Alpine Linux hosts.
- Installs Docker from system packages on Alpine.
- Creates runner files in `/srv/gitea_runner`.
- Stores runner state in `/srv/gitea_runner/data`.
- Mounts `/var/run/docker.sock` for rootful job containers.
- Renders `.env`, `config.yaml`, and `compose.yaml`.
- Runs `docker compose up -d` when rendered files change.

## Requirements

- Ansible 2.14 or newer.
- A Gitea runner registration token.
- A target Linux host supported by Docker.
- Root user on the target host.

## Quick Start

Install the role from Ansible Galaxy:

```bash
ansible-galaxy role install serega404.gitea_runner_docker
```

Then use it from your playbook:

```yaml
- name: Install Gitea Runner
  hosts: runners
  become: true
  roles:
    - role: serega404.gitea_runner_docker
      vars:
        gitea_runner_instance_url: https://git.example.com
        gitea_runner_registration_token: "{{ vault_gitea_runner_token }}"
```

Run:

```bash
ansible-playbook -i inventory.yml playbook.yml
```

## Repository Playbook

The repository already includes a ready-to-run playbook for the host from
`inventory.yml`:

```bash
ansible-playbook playbooks/install.yml
```

The playbook asks for:

- runner name and `/srv` directory postfix, default `main`;
- runner version, default `2.0`;
- Gitea instance URL;
- runner registration token.

If you accept `main` as the name, files are written to
`/srv/gitea_runner_main`. The version prompt accepts `2.0` and maps it to the
Docker image tag `docker.io/gitea/runner:2.0.0`.

You can also use the role from your own playbook:

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

Run:

```bash
ansible-playbook -i inventory.yml playbook.yml
```

## Main Variables

| Variable                             | Default                        | Description                                                             |
| ------------------------------------ | ------------------------------ | ----------------------------------------------------------------------- |
| `gitea_runner_dir`                   | `/srv/gitea_runner`            | Directory for Compose files and persistent data.                        |
| `gitea_runner_image`                 | `docker.io/gitea/runner:2.0.0` | Runner image.                                                           |
| `gitea_runner_instance_url`          | `""`                           | Gitea URL, required variable.                                           |
| `gitea_runner_registration_token`    | `""`                           | Registration token, required variable. Tasks that use it have `no_log`. |
| `gitea_runner_name`                  | `{{ inventory_hostname }}`     | Runner name in Gitea.                                                   |
| `gitea_runner_labels`                | Ubuntu/Node defaults           | Labels advertised by the runner to Gitea.                               |
| `gitea_runner_install_docker`        | `true`                         | Install Docker if `docker --version` does not work.                     |
| `gitea_runner_manage_docker_service` | `true`                         | Enable and start the Docker service.                                    |
| `gitea_runner_manage_compose`        | `true`                         | Run `docker compose up -d` through the handler.                         |
| `gitea_runner_docker_socket`         | `/var/run/docker.sock`         | Host Docker socket mounted into the runner container.                   |
| `gitea_runner_compose_project_name`  | `gitea_runner`                 | Compose project name.                                                   |
| `gitea_runner_restart_policy`        | `unless-stopped`               | Runner container restart policy.                                        |
| `gitea_runner_cache_enabled`         | `false`                        | Enable cache settings and port publishing.                              |
| `gitea_runner_cache_host`            | `""`                           | Host/IP reachable by job containers when cache is enabled.              |
| `gitea_runner_cache_port`            | `8088`                         | Cache port.                                                             |
| `gitea_runner_cache_dir`             | `""`                           | Cache directory inside the runner container.                            |
| `gitea_runner_config`                | `{}`                           | Recursive override for the generated `config.yaml`.                     |
| `gitea_runner_config_raw`            | `""`                           | Full raw `config.yaml`; when set, the template is not generated.        |
| `gitea_runner_environment_extra`     | `{}`                           | Extra environment variables written to `.env`.                          |

The included `playbooks/install.yml` derives these variables from the answers:

- `gitea_runner_dir`: `/srv/gitea_runner_<normalized_name>`;
- `gitea_runner_name`: entered runner name;
- `gitea_runner_image`: `docker.io/gitea/runner:<version>`, where `2.0` is
  normalized to `2.0.0`;
- Compose project and container name: `gitea_runner_<normalized_name>`.

## Cache

! Not tested.

Cache support is disabled by default. To enable it, set a host address that job
containers can reach:

```yaml
gitea_runner_cache_enabled: true
gitea_runner_cache_host: 192.168.1.10
gitea_runner_cache_port: 8088
```

## License

[MIT](./LICENSE)
