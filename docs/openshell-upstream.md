# OpenShell Upstream Maintenance

This document records how `githubixx.ai.openshell` relates to NVIDIA's upstream OpenShell installer. It is a maintenance guide, not a copy of upstream `install.sh`.

## Upstream Baseline

- Repository: <https://github.com/NVIDIA/OpenShell>
- Installer: <https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh>
- Last reviewed release: `v0.0.92`
- Role default: `openshell_version: "0.0.92"`
- Supported role platform: Linux x86_64 on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux.

The role downloads the pinned CLI and gateway release artifacts directly from GitHub Releases. The archive checksums in [the role defaults](../roles/openshell/defaults/main.yml) must match the configured release.

## Intentional Differences

The role uses upstream artifacts, but does not reproduce `install.sh` exactly.

- It does not invoke or vendor `install.sh`.
- It does not use Debian, RPM, Homebrew, or other package-manager installation paths.
- It installs `openshell` and `openshell-gateway` for `openshell_user` in `~/.local/bin` by default.
- It creates and manages a systemd user service for the local gateway.
- It supports Docker as the compute driver and validates Docker socket access for `openshell_user`.
- It does not install, configure, or grant access to Docker.
- It does not support Podman, Kubernetes, MicroVM, remote gateway, Windows, macOS, or aarch64 targets.

## Upstream Review Checklist

Review the current upstream installer and the target release whenever NVIDIA changes `install.sh` or when updating `openshell_version`.

1. Compare artifact names, target triples, supported architectures, and archive contents with the role defaults.
2. Fetch release checksums from GitHub Releases and update the CLI and gateway checksums together with the version.
3. Check for changes to CLI or gateway runtime requirements, especially GNU libc and system dependencies.
4. Check whether gateway configuration, certificates, registration, or service lifecycle behavior has changed.
5. Check Docker driver defaults, socket paths, networking requirements, and any added driver support.
6. Compare upstream supported operating systems with the role's supported platform contract.
7. Update role defaults, tasks, templates, README, Molecule verification, and changelog entries as needed.
8. Run `molecule test` for the default scenario before publishing an updated collection version.

## Triggering a Review

A request such as "NVIDIA changed OpenShell `install.sh`; review the role" is enough to begin this comparison. Include the upstream release or commit when available so the review can be tied to an exact version.
