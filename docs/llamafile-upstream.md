# llamafile Upstream Maintenance

This document records how `githubixx.ai.llamafile` relates to Mozilla's pre-built llamafile catalog. It is a maintenance guide, not a copy of upstream download or launch scripts.

## Upstream Baseline

- Pre-built catalog: <https://docs.mozilla.ai/llamafile/getting-started/pre-built-llamafiles>
- Catalog repository: <https://huggingface.co/mozilla-ai/llamafile_0.10>
- Supported upstream series: `0.10.*`
- Role catalog revision: `132715bf3442e0e5ef8bd532a9ea9c5eb10030bd`
- Supported role platform: Linux x86_64 on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux.

The upstream artifacts are portable APE/MBR-format executables. They can run from an interactive shell, but systemd cannot execute them directly. The role uses a shell wrapper for server-mode `ExecStart` commands and an executable shell launcher for combined mode.

## Intentional Differences

The role uses fixed upstream artifacts but does not install or reproduce upstream tooling.

- It supports only the `0.10.*` pre-built catalog and uses an immutable Hugging Face revision for built-in entries.
- It requires a SHA-256 checksum for every built-in or custom artifact.
- It does not dynamically select a latest model or catalog revision.
- It does not install GPU drivers, CUDA, ROCm/HIP, Vulkan packages, firewall rules, TLS, authentication, or reverse proxies.
- It installs model bundles under `/opt/llamafile/models` only after checksum verification.
- It supports `server` mode as a systemd service and `combined` mode as an interactive terminal launcher; combined mode is intentionally never a systemd service.
- It does not estimate model memory requirements or accept model licenses on the operator's behalf.

## Upstream Review Checklist

Review the upstream catalog whenever Mozilla changes the `0.10.*` pre-built model list, artifact formats, checksums, or supported launch options.

1. Confirm the upstream release still belongs to the supported `0.10.*` series.
2. Select and pin an immutable Hugging Face revision in `llamafile_catalog_revision`.
3. For every changed or added catalog entry, verify its filename, URL, and upstream LFS SHA-256 checksum.
4. Verify that the bundle runs on each supported distribution and that server-mode bundles still require the systemd shell wrapper.
5. Review changes to `--server`, `--jinja`, `--gpu`, `-ngl`, context, thread, and health-endpoint behavior.
6. Review accelerator support without adding driver or toolkit installation outside the role's scope.
7. Update defaults, tasks, templates, README, Molecule verification, this guide, collection README, Galaxy metadata, and changelog entries as needed.
8. Run `ansible-lint` and `molecule test` for the default scenario before publishing an updated collection.

## Triggering a Review

A request such as "Mozilla updated the pre-built llamafile catalog; review the role" is sufficient to begin this comparison. Include the intended catalog revision or artifact URL when available so the review can be tied to immutable upstream content.
