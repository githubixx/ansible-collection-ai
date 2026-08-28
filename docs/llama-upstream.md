# llama.cpp Upstream Maintenance

This document records how `githubixx.ai.llama` relates to `https://llama.app/install.sh`. It is a maintenance guide, not a copy of the upstream installer.

## Upstream Baseline

- Installer: <https://llama.app/install.sh>
- Artifact bucket: <https://huggingface.co/buckets/ggml-org/install.sh>
- Role default release identifier: `b10612`
- Supported normal installation platform: Linux x86_64 on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux.

The upstream installer identifies Linux architecture, then tries CUDA, ROCm, Vulkan, and CPU in that order. It downloads small native probe helpers and chooses a feature-specific `llama-app.zst` executable. On macOS it separately selects a supported Metal binary.

## Version Selection

Use the release identifier published by the installer artifact bucket, not a GitHub release tag by itself. Query the current installer-supported version with:

```sh
curl -fsSL https://huggingface.co/buckets/ggml-org/install.sh/resolve/latest
```

At the time of this update, that endpoint returns `b10612`. GitHub can publish a newer `b...` pre-release or stable `v...` tag before the corresponding installer probe helpers and feature-specific binaries are available in the bucket. Before updating `llama_version`, confirm that `llama-probe` can download the required helper and selected artifact for every supported hardware family.

## Repeatable Installer Comparison

The committed [installer archive](../roles/llama/archive/install.sh) is the
baseline for every review. It is an unmodified copy of
`https://llama.app/install.sh` verified on 2026-08-28 with SHA-256:

```text
cccdfcbd1b55bf6003ac3037588c9f5b3b79aa0a75fe991e97bb218ccdb55e4d
```

`llama.app/install.sh` is not tracked in the public
[`ggml-org/llama.cpp`](https://github.com/ggml-org/llama.cpp) repository. The
repository contains the `app/` C++ sources but no production `install.sh`, so
this archive, rather than llama.cpp Git history, preserves the role's
installer baseline.

Compare the committed baseline with the current installer from the repository
root:

```sh
curl -fsSL https://llama.app/install.sh -o /tmp/llama-app-install.sh
sha256sum roles/llama/archive/install.sh /tmp/llama-app-install.sh
diff -u roles/llama/archive/install.sh /tmp/llama-app-install.sh
curl -fsSL https://huggingface.co/buckets/ggml-org/install.sh/resolve/latest
```

If the files differ, classify the functional differences below, implement and
verify any required role changes, then refresh the committed archive as part of
the same pull request. Refresh the archive even when the review concludes that
no role change is needed; this records that the new installer was deliberately
reviewed. Do not refresh it when the review is incomplete or validation fails.

```sh
cp /tmp/llama-app-install.sh roles/llama/archive/install.sh
sha256sum roles/llama/archive/install.sh
git diff -- roles/llama/archive/install.sh docs/llama-upstream.md
```

Update the verification date and SHA-256 above whenever the archive is
refreshed. The archive's Git commit then provides the precise prior baseline
for the next review.

Do not treat a changed installer hash alone as a role update. Classify every functional diff using this table:

| Upstream installer change | Role impact | Files to review |
| --- | --- | --- |
| Bucket name, release lookup, architecture or OS mapping | Update only when the role's supported platforms or artifact base URL must change. | `defaults/main.yml`, `tasks/main.yml`, `tasks/probe.yml`, this guide |
| Backend preference, helper name/path, feature-code output, archive format, or artifact path | Update the opt-in probe so it selects the same artifact as upstream. | `tasks/probe.yml`, `defaults/main.yml`, role README, Molecule |
| Decompression command or artifact installation behavior | Update pinned installation tasks and test the affected distributions. | `tasks/main.yml`, Molecule |
| `llama version` output or installer version-match rules | Update installed-version parsing and verifier assertions together. | `tasks/main.yml`, `molecule/default/verify.yml` |
| `llama serve` startup arguments, environment variables, router API, or health endpoint behavior | Update service rendering, defaults, validation, and API verification as needed. | `templates/llama.service.j2`, `defaults/main.yml`, `tasks/main.yml`, README, Molecule |
| User-local install paths, shell profile changes, symlink migration, `SKIP_*`, or installer-only download conveniences | No role change unless the pinned system-service contract is intentionally expanded. Record the difference here if it affects future reviews. | This guide only |

The current role deliberately differs from upstream in the final category. It installs a verified binary into `/usr/local/bin` and runs it as a dedicated systemd account, rather than modifying a user's `~/.local/bin`, shell configuration, or installer cache directory.

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

Review the current upstream installer and target artifact whenever llama.app changes `install.sh` or a new installer-supported version is adopted.

1. Query the artifact bucket's `latest` endpoint and confirm its identifier is the intended update target.
2. Compare the backend priority, platform/architecture paths, helper names, compression format, and artifact naming with the probe tasks.
3. Run `llama-probe` on each intended hardware family and retain the printed artifact URL and SHA-256 values.
4. Verify the selected executable's `llama version` output before updating `llama_version`.
5. Review upstream `llama serve` environment variables and command-line argument changes, especially router, model cache, API, authentication, and accelerator settings.
6. Check whether runtime dependencies or support boundaries changed for the selected accelerator artifact.
7. Update defaults, tasks, template, README, Molecule configuration, this guide, and the changelog together.
8. Run `ansible-lint` and `molecule test` for the default scenario before publishing.

## Triggering a Review

A request such as “llama.app changed `install.sh`; review the role” is sufficient to begin this comparison. Include the upstream release identifier or installer revision when available so the review can be tied to a specific artifact layout.
