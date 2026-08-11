# githubixx.ai.ollama

Installs the Ollama service from a pinned, SHA-256-verified GitHub release archive. The role does not invoke Ollama's `install.sh`, install GPU drivers, or alter firewall rules.

## Requirements

- Ansible Core 2.15 or later.
- Linux x86_64 running Ubuntu 24.04, Ubuntu 26.04, or Arch Linux.
- systemd.
- Network access to GitHub Releases during installation.

Ollama uses already-installed supported NVIDIA or AMD acceleration when available. This role does not install, configure, or validate GPU drivers. CPU inference is supported.

## Role variables

All public variables use the `ollama_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `ollama_version` | `0.32.9` | Pinned Ollama release. |
| `ollama_user` | `ollama` | Dedicated system service account. |
| `ollama_host` | `127.0.0.1:11434` | API bind address. |
| `ollama_api_url` | `http://{{ ollama_host }}` | Loopback API URL for role readiness checks. |
| `ollama_service_environment` | Default `HOME`, `OLLAMA_HOST`, and `OLLAMA_MODELS` values | Ordered systemd environment entries, each written as an `Environment=` directive. |
| `ollama_service_environmentfile` | Empty | Optional environment file, written as an `EnvironmentFile=` directive when set. |
| `ollama_state_directory` | `/usr/share/ollama/.ollama` | Model and service state directory. |
| `ollama_service_enabled` | `true` | Enable the system service. |
| `ollama_service_started` | `true` | Start the system service. |

The default checksum corresponds to the v0.32.9 Linux x86_64 base archive. Override the version, archive URL, and checksum together only after verifying the upstream release checksum.

See the [upstream maintenance guide](../../docs/ollama-upstream.md) when reviewing a new Ollama release or changes to upstream `install.sh`.

## Example

```yaml
- name: Install Ollama
  hosts: inference_hosts
  become: true
  roles:
    - role: githubixx.ai.ollama
      vars:
        ollama_host: "127.0.0.1:11434"
```

The role installs the CLI at `/usr/local/bin/ollama`, runtime libraries under `/usr/local/lib/ollama`, and models under `/usr/share/ollama/.ollama/models`. It manages `ollama.service` as the dedicated `ollama` system user.

## Localhost example

Create `install-ollama.yml`:

```yaml
---
- name: Install Ollama locally
  hosts: localhost
  connection: local
  become: true
  collections:
    - githubixx.ai
  roles:
    - role: ollama
```

Run it with:

```sh
ansible-playbook install-ollama.yml --ask-become-pass
```

## Service environment

Set `ollama_service_environment` to replace the unit's complete environment list. Each item is rendered as one quoted `Environment=` line, so use `KEY=value` strings e.g.:

```yaml
ollama_user: ouser
ollama_group: "{{ ollama_user }}"
ollama_home: "/home/{{ ollama_user }}"
ollama_state_directory: "{{ ollama_home }}/.ollama"
ollama_api_url: "http://127.0.0.1:11434"
ollama_service_environmentfile: "{{ ollama_home }}/.secrets/ollama.systemd"
ollama_service_environment:
  - "HOME={{ ollama_home }}"
  - "PATH={{ ollama_home }}/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
  - "OLLAMA_HOST=0.0.0.0:11434"
  - "OLLAMA_CONTEXT_LENGTH=8192"
  - "OLLAMA_EXPERIMENT=client2"
  - "OLLAMA_MODELS={{ ollama_state_directory }}/models"
```

Systemd does not expand `$HOME` in `Environment=` or `EnvironmentFile=` directives. Use absolute paths in values and file paths. The environment file must use systemd's `KEY=value` format and must be readable by the `ollama` service user.

## API exposure

The default API bind address is loopback-only. Setting `ollama_host` to a non-loopback address exposes Ollama without configuring TLS, authentication, or firewall rules. Configure those controls separately before exposing the service.

## Testing

The default Molecule scenario uses Vagrant with libvirt and verifies Ubuntu 24.04, Ubuntu 26.04, and Archlinux. It pulls and generates with `gemma3:270m`; each instance has 4096 MiB of memory and a 24 GiB root volume to make CPU inference reliable.
