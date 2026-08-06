# llama.cpp Upstream Maintenance

This document records how `githubixx.ai.llama` relates to `https://llama.app/install.sh`. It is a maintenance guide, not a copy of the upstream installer.

## Upstream Baseline

- Installer: <https://llama.app/install.sh>
- Artifact bucket: <https://huggingface.co/buckets/ggml-org/install.sh>
- Role default release identifier: `b10107`
- Supported normal installation platform: Linux x86_64 on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux.

The upstream installer identifies Linux architecture, then tries CUDA, ROCm, Vulkan, and CPU in that order. It downloads small native probe helpers and chooses a feature-specific `llama-app.zst` executable. On macOS it separately selects a supported Metal binary.

## Intentional Differences

The role uses upstream artifacts but does not reproduce the installer during normal provisioning.

- It requires an explicit `llama_artifact_url` and matching `llama_artifact_checksum`.
- It does not invoke or vendor `install.sh`.
- It does not dynamically select a latest release.
- It does not probe hardware, install GPU drivers, CUDA, ROCm, Vulkan, or vendor toolkits during normal execution.
- It creates a dedicated `llama` system user and shared model cache, then manages one or more `llama serve` systemd units.
- It defaults its service to loopback and does not configure TLS, reverse proxies, authentication, or firewall rules.
- It does not manage model downloads or model files; official environment variables and raw server arguments remain user-configurable.

## Optional Probe

The `llama-probe` tag is an explicit diagnostic preflight:

```sh
ansible-playbook install-llama.yml --tags llama-probe
```

It follows the upstream backend preference order, runs the downloaded upstream probe helpers, downloads the selected compressed binary temporarily, calculates its SHA-256, prints the role variable values, and cleans up. It does not install the binary or alter persistent configuration.

This remains a distinct trust boundary because it executes native upstream probe programs. Pin `llama_probe_version`, review the result, and commit the emitted artifact URL and checksum into inventory before normal installation. If upstream does not publish checksums for a helper, the helper itself cannot receive the same integrity guarantee as the final pinned artifact.

## Upstream Review Checklist

Review the current upstream installer and target artifact whenever llama.app changes `install.sh` or a new release is adopted.

1. Compare the backend priority, platform/architecture paths, helper names, compression format, and artifact naming with the probe tasks.
2. Run `llama-probe` on each intended hardware family and retain the printed artifact URL and SHA-256 values.
3. Verify the selected executable's `llama version` output before updating `llama_version`.
4. Review upstream `llama serve` environment variables and command-line argument changes, especially router, model cache, API, authentication, and accelerator settings.
5. Check whether runtime dependencies or support boundaries changed for the selected accelerator artifact.
6. Update defaults, tasks, template, README, Molecule configuration, this guide, and the changelog together.
7. Run `ansible-lint` and `molecule test` for the default scenario before publishing.

## Triggering a Review

A request such as “llama.app changed `install.sh`; review the role” is sufficient to begin this comparison. Include the upstream release identifier or installer revision when available so the review can be tied to a specific artifact layout.
