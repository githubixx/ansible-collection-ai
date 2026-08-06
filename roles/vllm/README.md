# githubixx.ai.vllm

Installs vLLM from a pinned PyPI package in a `uv`-managed Python 3.12 virtual environment and runs its OpenAI-compatible API server as a dedicated systemd service. The role does not install `uv`, GPU drivers, CUDA, ROCm, reverse proxies, TLS certificates, or firewall rules.

## Requirements

- Ansible Core 2.15 or later.
- Linux x86_64 running Ubuntu 24.04, Ubuntu 26.04, or Arch Linux.
- systemd.
- `uv` installed and available on the managed host PATH. The role uses it to obtain Python 3.12 when needed but does not install or update `uv`.
- Network access to PyPI, Astral's Python distributions, and, for AMD, the vLLM ROCm wheel index during installation.
- A configured and supported GPU runtime:
  - NVIDIA: a driver compatible with the CUDA PyTorch backend chosen by `uv --torch-backend=auto`.
  - AMD: ROCm 7.0 and `glibc >= 2.35`.

The role does not validate GPU availability during provisioning. The selected accelerator must match the preconfigured vendor runtime.

## Role variables

All public variables use the `vllm_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `vllm_version` | `0.26.0` | Pinned vLLM PyPI release. |
| `vllm_package` | `vllm` | PyPI package name. |
| `vllm_uv_executable` | `uv` | Preinstalled `uv` executable. |
| `vllm_python_version` | `3.12` | Python runtime requested from `uv`. |
| `vllm_accelerator` | `nvidia_cuda` | Supported accelerator profile: `nvidia_cuda` or `amd_rocm`. |
| `vllm_accelerator_profiles` | CUDA auto-selection and ROCm wheel index profiles | Profile-specific `uv pip install` arguments. |
| `vllm_install_arguments` | Selected profile arguments | Override installation arguments for a compatible vendor runtime. |
| `vllm_install_directory` | `/opt/vllm` | Virtual environment parent directory. |
| `vllm_home` | `/var/lib/vllm` | Dedicated service account home and persistent state base directory. |
| `vllm_cache_directory` | `/var/lib/vllm/cache` | Persistent vLLM and Hugging Face cache base directory. |
| `vllm_models_directory` | `/var/lib/vllm/models` | Operator-managed local model directory. It is not passed to vLLM by default. |
| `vllm_user` | `vllm` | Dedicated system service account. |
| `vllm_model` | `Qwen/Qwen2.5-1.5B-Instruct` | Hugging Face model ID or local model path served by vLLM. |
| `vllm_host` | `127.0.0.1` | API bind address. |
| `vllm_port` | `8000` | API port. |
| `vllm_api_url` | `http://{{ vllm_host }}:{{ vllm_port }}` | Loopback readiness URL used by the role. |
| `vllm_service_environment` | Default home, PATH, cache, config, and Hugging Face settings | Ordered systemd `Environment=` entries. |
| `vllm_service_environmentfile` | Empty | Optional operator-managed systemd environment file. |
| `vllm_service_arguments` | Host and port arguments | Additional arguments passed to `vllm serve`. |
| `vllm_service_enabled` | `true` | Enable the system service. |
| `vllm_service_started` | `true` | Start the system service. |

Changing `vllm_accelerator` rebuilds the virtual environment. This prevents incompatible CUDA and ROCm Python dependencies from being reused across installations.

## Examples

```yaml
- name: Install vLLM with NVIDIA CUDA
  hosts: vllm_hosts
  become: true
  roles:
    - role: githubixx.ai.vllm
      vars:
        vllm_accelerator: nvidia_cuda
```

```yaml
- name: Install vLLM with AMD ROCm
  hosts: vllm_hosts
  become: true
  roles:
    - role: githubixx.ai.vllm
      vars:
        vllm_accelerator: amd_rocm
```

The default deployment starts `vllm serve Qwen/Qwen2.5-1.5B-Instruct` and exposes the OpenAI-compatible API at `http://127.0.0.1:8000/v1`. It implements endpoints including `/v1/models`, `/v1/completions`, and `/v1/chat/completions`.

## Credentials and gated models

Set `vllm_service_environmentfile` to an existing operator-managed environment file to supply secrets without placing them in playbook output or the generated systemd unit. Restrict the file to root and the vLLM service group, for example with mode `0640` and owner `root:vllm`.

```ini
HF_TOKEN=replace-with-a-hugging-face-token
VLLM_API_KEY=replace-with-a-long-random-api-key
```

`HF_TOKEN` is needed for gated Hugging Face models. `VLLM_API_KEY` makes vLLM require a matching bearer token. The role does not create this file or validate its values.

## Exposure

The default bind address is loopback-only. Changing `vllm_host` to a non-loopback address exposes the API without configuring TLS, reverse-proxy authentication, firewall rules, or rate limits. Configure those controls separately, set `VLLM_API_KEY`, and use an operator-managed environment file before exposing the service.

## Model storage

vLLM downloads Hugging Face models into `/var/lib/vllm/cache/huggingface` by default. This must be persistent storage with enough capacity for the selected model and its runtime artifacts. Set `vllm_model` to a path below `vllm_models_directory` or another accessible local path to serve a pre-downloaded model.

## Testing

The default Molecule scenario validates package installation, virtual-environment ownership, accelerator-profile reconciliation, and rendered systemd units on Ubuntu 24.04, Ubuntu 26.04, and Arch Linux. It leaves the service disabled and stopped because the Vagrant guests do not provide NVIDIA or AMD GPUs.

On each prepared GPU host, enable and start the service, then verify `GET /health`, `GET /v1/models`, and a small OpenAI-compatible completion or chat request. GPU-backed tests are required to confirm a model and the installed vendor runtime work together.
