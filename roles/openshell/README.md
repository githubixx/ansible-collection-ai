# githubixx.ai.openshell

Installs the NVIDIA OpenShell CLI, standalone local gateway, and policy prover from pinned, SHA-256-verified GitHub release archives. The role does not use NVIDIA's `install.sh`, which only supports Debian and RPM package managers.

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
| `openshell_version` | `0.1.2` | Pinned OpenShell CLI, gateway, and prover release. |
| `openshell_user` | `ansible_user` | Non-root user owning the local gateway. |
| `openshell_bin_directory` | `~/.local/bin` | Executable directory owned by `openshell_user`. |
| `openshell_compute_driver` | `docker` | The selected gateway driver. Only Docker is supported initially. |
| `openshell_default_image` | `nvcr.io/nvidia/base/ubuntu:24.04` | Workload image seeded in new gateway configuration. |
| `openshell_gateway_bind_address` | `127.0.0.1:17670` | Gateway bind address and port. |
| `openshell_gateway_service_enabled` | `true` | Enable the user service. |
| `openshell_gateway_service_started` | `true` | Start the user service and register the local gateway. |

The default checksums correspond to v0.1.2 x86_64 release artifacts. Override the version, archive URLs, and all three checksums together only after verifying the upstream release checksums.

Check the [latest OpenShell release](https://github.com/NVIDIA/OpenShell/releases/latest) for the current version and SHA-256 checksums.

See the [upstream maintenance guide](../../docs/openshell-upstream.md) when reviewing a new OpenShell release or changes to NVIDIA's `install.sh`.

## Upgrading From 0.0.x

Local OpenShell 0.0.x installations cannot be upgraded in place to 0.1.x. This role rejects legacy or unrecognized installed binaries, orphaned runtime state, and retained configuration not using schema v2 before changing installation files. It does not delete sandboxes, rewrite databases, or automatically migrate user configuration.

Follow the [upstream 0.1.0 migration guide](https://docs.nvidia.com/openshell/latest/upgrade/0-1-0.html) before applying this role to an existing installation:

1. Export provider profiles with the old CLI and back up required sandbox files, gateway data, configuration, credentials, and encryption material. For a WAL-mode SQLite database use SQLite's backup tooling, not a copy of the database file alone.
2. Coordinate a downtime window, remove legacy sandboxes with the old CLI, stop the gateway, and clean up the old runtime and installation as described upstream. Keep backups outside the installation and state directories. Upgrade all communicating components together; mixed 0.0.x and 0.1.x peers are unsupported.
3. Install the new release into a clean local runtime. Manually migrate any retained `gateway.toml` to schema version 2 and run `openshell-gateway config preflight --path ~/.config/openshell/gateway.toml`. Edited configuration is preserved, not overwritten.
4. Import the saved provider profiles, recreate sandboxes and credentials as required, and update workflows using removed managed inference routes or old API/policy fields.

Fresh gateway configuration selects Docker with schema version 2 and the minimal `nvcr.io/nvidia/base/ubuntu:24.04` workload image. It includes no agent CLI or image-baked policy; supply a fully qualified workload image containing the required tools. The gateway's sandbox and supervisor runtime images default to its own release. Startup preflight validates configuration before generating certificates or starting the gateway.

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

The role creates the selected user's `~/.config/openshell/gateway.toml` only when it does not already exist. It preserves an existing user-managed file. The CLI, gateway, and prover executables are installed together in that user's `~/.local/bin`. The systemd unit is managed by the role at `~/.config/systemd/user/openshell-gateway.service`.

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
