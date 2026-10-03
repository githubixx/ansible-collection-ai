# githubixx.ai.openwebui

Installs Open WebUI from a pinned PyPI package in a `uv`-managed Python 3.11 virtual environment and runs it as a dedicated systemd service. The role does not install Docker, GPU drivers, CUDA, ROCm, PostgreSQL, Redis, reverse proxies, TLS certificates, or firewall rules.

## Requirements

- Ansible Core 2.15 or later.
- Linux x86_64 running Ubuntu 24.04, Ubuntu 26.04, or Arch Linux.
- systemd.
- `uv` installed and available on the managed host PATH. The role uses it to obtain Python 3.11 when needed but does not install or update `uv`.
- Network access to PyPI and Astral's Python distributions during installation.
- Network access to Hugging Face when startup needs to download embedding models.
- An environment file containing a persistent `WEBUI_SECRET_KEY`.

The default SQLite deployment needs locally attached storage. Do not put its data directory on NFS, CIFS/SMB, or other network filesystems.

## Role variables

All public variables use the `openwebui_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `openwebui_version` | `0.11.4` | Pinned Open WebUI PyPI release. |
| `openwebui_package` | `open-webui[all]` | Package name and dependency profile installed with `uv`. The `all` extra includes PostgreSQL support. |
| `openwebui_uv_executable` | `uv` | Preinstalled `uv` executable. |
| `openwebui_python_version` | `3.11` | Python runtime requested from `uv`. |
| `openwebui_install_directory` | `/opt/openwebui` | Virtual environment location. |
| `openwebui_home` | `/var/lib/openwebui` | Dedicated service account home and state base directory. |
| `openwebui_data_directory` | `/var/lib/openwebui/data` | Local uploads, cache, vector store, and SQLite database directory. |
| `openwebui_user` | `openwebui` | Dedicated system service account. |
| `openwebui_database_backend` | `sqlite` | Supported database mode: `sqlite` or `postgresql`. |
| `openwebui_host` | `127.0.0.1` | HTTP bind address. |
| `openwebui_port` | `8080` | HTTP port. |
| `openwebui_api_url` | `http://{{ openwebui_host }}:{{ openwebui_port }}` | Loopback readiness URL used by the role. |
| `openwebui_api_retries` | `120` | Maximum readiness retries; first startup may download embedding models. |
| `openwebui_api_retry_delay` | `5` | Seconds between readiness attempts; each HTTP request has a five-second timeout. |
| `openwebui_service_environment` | Default home, PATH, cache, config, data, and SQLite database settings | Ordered systemd `Environment=` entries. |
| `openwebui_service_environmentfile` | `/etc/openwebui/openwebui.env` | Required absolute path to an operator-managed systemd environment file. |
| `openwebui_service_arguments` | Host and port arguments | Arguments passed to `open-webui serve`. |
| `openwebui_service_enabled` | `true` | Enable the system service. |
| `openwebui_service_started` | `true` | Start the system service. |

## Upgrades

Before changing the pinned release, stop the service and take a consistent backup of the database, data directory, and environment file, including the existing `WEBUI_SECRET_KEY`. Use the database's backup tooling for PostgreSQL. Keep the secret key unchanged and review upstream schema migrations; a package downgrade alone does not roll back a migrated database. Restore the matching database and state backup when rolling back.

Version 0.11.4 includes security fixes. Connections that forward browser cookies now require explicit cookie-forwarding opt-in. Personal Direct Integrations are hidden by default until an administrator enables them; existing connections continue working. Tools and functions must declare dependencies they import rather than relying on incidental packages such as `langchain-community`. Review upstream PostgreSQL knowledge-index remediation when applicable. Docker slim-image restrictions do not apply to this native `open-webui[all]` installation.

## Required environment file

Create the environment file before applying the role. It is not managed by this role so secrets do not enter playbook output or the generated unit file. Restrict it to root and the Open WebUI service group, for example with mode `0640` and owner `root:openwebui`.

```ini
WEBUI_SECRET_KEY=replace-with-a-persistent-random-secret
```

Generate a value with:

```sh
openssl rand -hex 32
```

Open WebUI uses this key to sign sessions and encrypt sensitive data. Keep it stable: rotating it invalidates existing sessions and can make encrypted values unavailable.

## Example

```yaml
- name: Install Open WebUI
  hosts: webui_hosts
  become: true
  roles:
    - role: githubixx.ai.openwebui
```

The role creates `/opt/openwebui/venv`, runs `openwebui.service` as `openwebui:openwebui`, and stores persistent local state under `/var/lib/openwebui/data`. The generated unit keeps service arguments and restart directives on separate lines and loads the required environment file before launching the application.

## PostgreSQL

Set `openwebui_database_backend: postgresql` and provide the existing database URL in the required environment file. The role does not install PostgreSQL, create a database or account, create extensions, or migrate existing SQLite data.

```yaml
- name: Install Open WebUI with PostgreSQL
  hosts: webui_hosts
  become: true
  roles:
    - role: githubixx.ai.openwebui
      vars:
        openwebui_database_backend: postgresql
```

```ini
WEBUI_SECRET_KEY=replace-with-a-persistent-random-secret
DATABASE_URL=postgresql://openwebui:encoded-password@db.example.net:5432/openwebui
```

URL-encode special characters in database credentials. `DATA_DIR` remains local service state for uploads and other application files even when PostgreSQL stores the main database. For multi-worker or multi-instance deployments, configure PostgreSQL, Redis, a shared-safe vector store, and Open WebUI migration ownership outside this role.

## Exposure

The default bind address is loopback-only. Changing `openwebui_host` to a non-loopback address exposes Open WebUI without configuring TLS, authentication, a reverse proxy, or firewall rules. Configure those controls separately before exposing the service.

## Testing

The default Molecule scenario verifies the native service on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux. On a clean guest it installs 0.11.0, seeds a SQLite database sentinel and a persistent data file, then upgrades to the pinned release. An interrupted fixture can resume without discarding its database or downloaded models. Verification checks retained state and the test-only secret, Python 3.11, installed package metadata and dependency consistency, systemd ownership/state, and the loopback UI. It does not configure a model provider, create an Open WebUI user, or provision external databases.
