# Ollama Upstream Maintenance

This document records how `githubixx.ai.ollama` relates to Ollama's upstream installer. It is a maintenance guide, not a copy of upstream `install.sh`.

## Upstream Baseline

- Repository: <https://github.com/ollama/ollama>
- Installer: <https://raw.githubusercontent.com/ollama/ollama/main/scripts/install.sh>
- Last reviewed release: `v0.32.9`
- Role default: `ollama_version: "0.32.9"`
- Supported role platform: Linux x86_64 on Ubuntu 24.04, Ubuntu 26.04, and Archlinux.

The role downloads `ollama-linux-amd64.tar.zst` directly from GitHub Releases. The checksum in [the role defaults](../roles/ollama/defaults/main.yml) must match the configured release.

## Intentional Differences

The role uses upstream artifacts, but does not reproduce `install.sh` exactly.

- It does not invoke or vendor `install.sh`.
- It does not install GPU drivers, ROCm, CUDA repositories, kernel headers, or GPU packages.
- It installs the base Linux x86_64 archive under `/usr/local`.
- It creates a dedicated `ollama` system user and manages a systemd service.
- It defaults the API to `127.0.0.1:11434` and does not configure TLS, authentication, reverse proxies, or firewall rules.
- It does not support arm64, JetPack, GPU-specific archives, Windows, macOS, or containers.

## Upstream Review Checklist

Review the current upstream installer and target release whenever Ollama changes `install.sh` or when updating `ollama_version`.

1. Compare artifact names, target triples, supported architectures, and archive contents with the role defaults.
2. Fetch the target archive checksum from GitHub Releases and update it together with the version.
3. Check for new extraction, runtime, or service prerequisites.
4. Check service account, model-directory, API binding, and environment variable behavior.
5. Review upstream GPU, ROCm, CUDA, and JetPack changes without adopting driver installation unintentionally.
6. Compare upstream supported operating systems with the role's supported platform contract.
7. Update role defaults, tasks, template, README, Molecule verification, and changelog entries as needed.
8. Run `molecule test` for the default scenario before publishing an updated collection version.

## Triggering a Review

A request such as "Ollama changed `install.sh`; review the role" is enough to begin this comparison. Include the upstream release or commit when available so the review can be tied to an exact version.
