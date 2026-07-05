# AGENTS.md

Guidance for agents working in this repository.

## Project Overview

This repository is a standalone Ansible role for installing Gitea Runner in
rootful Docker Compose mode.

The role:

- installs Docker only when it is missing;
- uses the official Docker convenience script on non-Alpine Linux;
- uses Alpine packages on Alpine;
- renders runner files under `/srv/gitea_runner` or
  `/srv/gitea_runner_<name>` when using `playbooks/install.yml`;
- persists runner state in the `data/` directory beside the Compose file;
- mounts `/var/run/docker.sock` into the runner container.

Treat the Docker socket mount as a privileged boundary. Jobs running on this
runner can effectively control Docker on the host.

## Important Files

- `defaults/main.yml`: public role variables and defaults.
- `tasks/main.yml`: role entrypoint and validation.
- `tasks/docker.yml`: Docker detection and install routing.
- `tasks/install_alpine.yml`: Alpine Docker package install path.
- `tasks/install_official.yml`: official `get.docker.com` install path.
- `tasks/setup.yml`: directory and template rendering.
- `handlers/main.yml`: `docker compose up -d` handler.
- `templates/*.j2`: `.env`, `config.yaml`, and `compose.yaml` templates.
- `playbooks/install.yml`: local convenience playbook with prompts and Python
  bootstrap.
- `inventory.yml`: local test/apply inventory.

## Local Apply Flow

Use the prepared playbook from the repository root:

```bash
ansible-playbook playbooks/install.yml
```

The playbook prompts for:

- runner name and `/srv` directory postfix, default `main`;
- runner version, default `2.0`;
- Gitea instance URL;
- runner registration token.

The prompt value `2.0` is normalized to image tag `2.0.0`.

The runner name is normalized for filesystem/container names and used in:

- `/srv/gitea_runner_<normalized_name>`;
- `gitea_runner_<normalized_name>` Compose project name;
- `gitea_runner_<normalized_name>` container name.

## Secrets

Never commit real Gitea runner registration tokens.

The role writes the token to `.env` on the target host with mode `0600`, and
token-related Ansible tasks use `no_log`.

Keep examples in docs and tests fake, for example:

```yaml
gitea_runner_registration_token: example-token
```

## Coding Rules

- Keep role defaults in `defaults/main.yml`.
- Keep distro-specific install behavior in separate task files.
- Prefer Ansible modules over shell.
- The official Docker install task is intentionally shell-based because the role
  requirement is `curl -sSL https://get.docker.com | sh`; keep the lint `noqa`
  there unless the install strategy changes.
- Use `ansible.builtin.*` FQCNs.
- Do not add external collection dependencies unless the role truly needs them.
- Do not store generated local Ansible artifacts. `.ansible/` and `.cache/` are
  ignored.

## Documentation Rules

- Update both `README.md` and `README.ru.md` for user-facing behavior changes.
- Mention security-sensitive defaults, especially rootful Docker socket access.
- If changing the default Gitea Runner image or version handling, update the
  Runner notes and examples.

## Validation Commands

Run these from the repository root:

```bash
ansible-playbook --syntax-check playbooks/install.yml
yamllint inventory.yml playbooks/install.yml defaults/main.yml tasks handlers meta
ANSIBLE_GALAXY_CACHE_DIR=/tmp/ansible-galaxy-cache ansible-lint
```

If Ansible tries to write under a read-only home directory in this environment,
use temp/cache paths under `/tmp`.

## Compatibility Notes

- The convenience playbook bootstraps `python3` before gathering facts, so it can
  run on minimal Alpine/Debian/Fedora/Ubuntu hosts.
- Alpine Docker install uses package names `docker` and `docker-cli-compose`.
- Non-Alpine Docker install assumes the official Docker convenience script
  supports the target distribution.
- The role default image is `docker.io/gitea/runner:2.0.0`.
