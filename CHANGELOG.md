# Changelog

All notable changes to this collection are documented in this file.

## 0.4.0

**Breaking change:** OpenShell 0.0.x installations cannot be upgraded in place to 0.1.2. Follow the [OpenShell migration instructions](roles/openshell/README.md#upgrading-from-00x) before upgrading.

- deps: update Ollama to v0.35.0
- deps: update ComfyUI to v0.38.0
- deps: update Open WebUI to v0.11.4
- deps: update vllm to v0.30.0
- deps: update llama.cpp to b11200
- deps: update OpenShell to v0.1.2
- deps: refresh the twelve existing llamafile 0.10.* entries at catalog revision `8c9e234f2068fa6a6f3ed171926b367e65cef160`, without adding models
- chore: synchronize release checksums and Molecule expectations; verify Python application dependency consistency
- fix(ComfyUI): remove torchaudio from built-in PyTorch profiles
- fix(ComfyUI): wait for HTTP readiness in Molecule verification
- fix(openwebui): correct systemd command newlines and environment-file syntax; provision ACL tooling in Molecule guests
- fix(openwebui): allow configurable first-start readiness retries for embedding-model downloads and resume interrupted upgrade fixtures safely
- fix(openwebui): detect the installed version from package metadata instead of the unsupported CLI `--version` option, preserving idempotence
- fix(vllm): align the AMD wheel index with ROCm 7.2.3
- fix(vllm): verify the AMD PyTorch backend using HIP and CUDA build metadata instead of requiring a `rocm` package-version suffix
- fix(llamafile): create temporary download scripts only for missing or checksum-mismatched artifacts, preserving idempotence
- feat(openshell): install the release-matched policy prover
- feat(openshell)!: require gateway configuration schema v2 and reject unsupported in-place upgrades from 0.0.x before modifying files; document the required clean operator-managed migration

## 0.3.0

- deps: update OpenShell to v0.0.116
- deps: update ComfyUI to v0.34.0
- deps: update Ollama to v0.33.1
- deps: update llama.cpp to b10612
- deps: update vllm to v0.28.0

## 0.2.3

- deps: update OpenShell to v0.0.104
- chore(openshell): retry transient GitHub release downloads

## 0.2.2

- fix(ComfyUI): fix `ExecStart` value in `comfyui.service.j2`
- fix(ComfyUI): ensure `.ansible` and `.ansible/tmp` directories exists
- deps: update ComfyUI to v0.32.0

## 0.2.1

- update Ollama to v0.12.3
- generate GitHub Release notes with `git-cliff`

## 0.2.0

- add the `githubixx.ai.llamafile` role for pinned pre-built llamafile 0.10.* model bundles.

## 0.1.6

- update `galaxy.yml`
- update `CHANGELOG`

## 0.1.5

- update ComfyUI to v0.31.0
- update OpenShell to version v0.0.101

## 0.1.4

- fix collection version parsing in the release workflow.

## 0.1.3

- add manual release workflow dispatch trigger.

## 0.1.2

- fix `release.yml`
- update `CHANGELOG`

## 0.1.1

- fix release.yml

## 0.1.0 - 2026-07-16

### Added

- initial `githubixx.ai.openshell` role for installing NVIDIA OpenShell on supported Linux hosts.
- initial `githubixx.ai.ollama` role for installing Ollama on supported Linux hosts.
- initial `githubixx.ai.llama` role for installing llama.cpp `llama serve` instances from pinned llama.app binaries.
- initial `githubixx.ai.openwebui` role for installing Open WebUI as a native systemd service.
- initial `githubixx.ai.vllm` role for installing vLLM as a native OpenAI-compatible systemd service.
