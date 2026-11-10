# OpenShell Upstream Maintenance

This document records how `githubixx.ai.openshell` relates to NVIDIA's upstream OpenShell installer. It is a maintenance guide, not a copy of upstream `install.sh`.

## Upstream Baseline

- Repository: <https://github.com/NVIDIA/OpenShell>
- Installer: <https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh>
- Last reviewed release: `v0.1.8` (2026-11-10)
- Role default: `openshell_version: "0.1.8"`
- Supported role platform: Linux x86_64 on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux.

The role downloads pinned CLI, gateway, and policy prover release artifacts directly from GitHub Releases. The archive checksums in [the role defaults](../roles/openshell/defaults/main.yml) must match the configured release.

The v0.1.8 archive names and executable layout remain unchanged, and the gateway's minimum GNU libc version remains 2.28. The CLI, gateway, and prover checksums match both their GitHub release asset digests and NVIDIA's corresponding checksum manifests. The role's schema-v2 Docker configuration and packaged user-service startup sequence remain compatible. Upstream documents Docker Engine 28.0 as the minimum for the Docker driver.

## Intentional Differences

The role uses upstream artifacts, but does not reproduce `install.sh` exactly.

- It does not invoke or vendor `install.sh`.
- It does not use Debian, RPM, Homebrew, or other package-manager installation paths.
- It installs `openshell`, `openshell-gateway`, and `openshell-prover` for `openshell_user` in `~/.local/bin` by default, matching the native package's sibling-binary layout.
- It creates and manages a systemd user service for the local gateway.
- It supports Docker as the compute driver and validates Docker socket access for `openshell_user`.
- It does not install, configure, or grant access to Docker.
- It does not support Podman, Kubernetes, MicroVM, remote gateway, Windows, macOS, or aarch64 targets.

## Upstream Review Checklist

Review the current upstream installer and the target release whenever NVIDIA changes `install.sh` or when updating `openshell_version`.

1. Compare artifact names, target triples, supported architectures, and archive contents with the role defaults.
2. Fetch release checksums from GitHub Releases and update the CLI, gateway, and prover checksums together with the version.
3. Check for changes to CLI or gateway runtime requirements, especially GNU libc and system dependencies.
4. Check whether gateway configuration, certificates, registration, or service lifecycle behavior has changed.
5. Check Docker driver defaults, socket paths, networking requirements, and any added driver support.
6. Compare upstream supported operating systems with the role's supported platform contract.
7. Update role defaults, tasks, templates, README, Molecule verification, and changelog entries as needed.
8. Run `molecule test` for the default scenario before publishing an updated collection version.

## Triggering a Review

The 0.1.8 review follows the tagged native package and service definitions. The gateway template uses schema v2 and Docker fields; startup runs `config preflight` and generates a local TLS bundle. The default local bundle enables mTLS user authentication; the role does not opt into unauthenticated access. Existing TOML remains operator-owned, so custom authentication and new preflight constraints require review before upgrading. In particular, v0.1.8 requires explicit placement metadata for static provider credentials outside the `Authorization` header and changes OCSF sandbox attribution fields. See the [role upgrade notes](../roles/openshell/README.md#upgrading-within-01x) for runtime and audit changes. Legacy binaries, orphaned state, and older configuration schemas require the [operator-managed clean migration](https://docs.nvidia.com/openshell/latest/upgrade/0-1-0.html). The role does not install the packaged MicroVM driver, which is outside its Docker-only scope.

A request such as "NVIDIA changed OpenShell `install.sh`; review the role" is enough to begin this comparison. Include the upstream release or commit when available so the review can be tied to an exact version.
