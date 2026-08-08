# githubixx.ai.openshell

Installs the NVIDIA OpenShell CLI and standalone local gateway from pinned, SHA-256-verified GitHub release archives. The role does not use NVIDIA's `install.sh`, which only supports Debian and RPM package managers.

## Requirements

- Ansible Core 2.15 or later.
- Linux x86_64 running Ubuntu 24.04, Ubuntu 26.04, or Arch Linux.
- systemd with a functioning user manager; the role enables lingering for the selected user.
- GNU libc 2.28 or later for `openshell-gateway`. The CLI is statically linked with musl and does not have this runtime requirement.
- Docker Engine installed and usable by `openshell_user`. You can use the [githubixx.docker](https://github.com/githubixx/ansible-role-docker) role to install Docker. This role validates Docker socket access but intentionally does not install or configure Docker. When a running systemd user manager lacks the user's Docker group, the role restarts that manager before starting the gateway.

NVIDIA officially supports Debian/Ubuntu host platforms. Arch Linux is covered by this role as a best-effort target and is tested separately; it is not in the upstream host support matrix.

## Role variables

All public variables use the `openshell_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `openshell_version` | `0.0.101` | Pinned OpenShell CLI and gateway release. |
| `openshell_user` | `ansible_user` | Non-root user owning the local gateway. |
| `openshell_bin_directory` | `~/.local/bin` | Executable directory owned by `openshell_user`. |
| `openshell_compute_driver` | `docker` | The selected gateway driver. Only Docker is supported initially. |
| `openshell_gateway_bind_address` | `127.0.0.1:17670` | Gateway bind address and port. |
| `openshell_gateway_service_enabled` | `true` | Enable the user service. |
| `openshell_gateway_service_started` | `true` | Start the user service and register the local gateway. |

The default checksums correspond to v0.0.101 x86_64 release artifacts. Override the version, archive URLs, and both checksums together only after verifying the upstream release checksums.

Check the [latest OpenShell release](https://github.com/NVIDIA/OpenShell/releases/latest) for the current version and SHA-256 checksums.

See the [upstream maintenance guide](../../docs/openshell-upstream.md) when reviewing a new OpenShell release or changes to NVIDIA's `install.sh`.

## Example

```yaml
- name: Install OpenShell
  hosts: workstations
  become: true
  roles:
    - role: githubixx.ai.openshell
      vars:
        openshell_user: developer
```

The role creates the selected user's `~/.config/openshell/gateway.toml` only when it does not already exist. It preserves an existing user-managed file. The CLI and gateway executables are installed in that user's `~/.local/bin`. The systemd unit is managed by the role at `~/.config/systemd/user/openshell-gateway.service`.

## Localhost example

After installing Docker and granting your user Docker socket access, create `install-openshell.yml`:

```yaml
---
- name: Install OpenShell locally
  hosts: localhost
  connection: local
  become: true
  collections:
    - githubixx.ai
  roles:
    - role: openshell
      vars:
        openshell_user: "{{ ansible_user_id }}"
```

Run it with:

```sh
ansible-playbook install-openshell.yml --ask-become-pass
```

To deliberately install binaries system-wide instead, override the destination and ownership values together:

```yaml
openshell_bin_directory: /usr/local/bin
openshell_owner: root
openshell_group: root
```

## Compute driver

The gateway configuration defaults to Docker and listens on `127.0.0.1:17670`. Install and configure Docker Engine independently, including granting `openshell_user` access to the Docker socket, typically through the `docker` group. Podman, Kubernetes, MicroVM, remote gateways, Windows, macOS, and aarch64 are outside the initial role scope.

## Testing

The default Molecule scenario uses Vagrant with libvirt and verifies Ubuntu 24.04, Ubuntu 26.04, and Arch Linux. It runs `prepare`, `converge`, `idempotence`, and `verify`. The scenario installs Docker through the local `githubixx.docker` role, then grants the Molecule connection user access to the Docker group.
