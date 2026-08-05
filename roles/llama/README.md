# githubixx.ai.llama

Installs `llama serve` from a pinned, SHA-256-verified prebuilt `llama.app` binary and manages one or more systemd services. The role does not invoke the upstream installer during normal provisioning, install GPU drivers, download models, or alter firewall rules.

## Requirements

- Ansible Core 2.15 or later.
- Linux x86_64 running Ubuntu 24.04, Ubuntu 26.04, or Arch Linux.
- systemd.
- Network access to the configured artifact URL during installation.
- A pinned `llama_artifact_url` and matching `llama_artifact_checksum`.

The normal installation contract is CPU-only. The `llama-probe` preflight can recommend a CUDA, ROCm, Vulkan, or CPU artifact, but installing an accelerator-specific artifact and its runtime prerequisites remains the operator's responsibility.

## Role variables

All public variables use the `llama_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `llama_version` | `b10107` | Pinned upstream llama.app release identifier. |
| `llama_artifact_url` | Empty | Required URL to a selected `llama-app.zst` executable. |
| `llama_artifact_checksum` | Empty | Required SHA-256 checksum in `sha256:<digest>` format. |
| `llama_user` | `llama` | Shared dedicated system service account. |
| `llama_group` | `llama` | Shared dedicated system group. |
| `llama_home` | `/var/lib/llama` | Shared service home directory. |
| `llama_cache_directory` | `{{ llama_home }}/.cache/llama.cpp` | Shared llama.cpp model cache. |
| `llama_instances` | One loopback router instance | List of server instances described below. |
| `llama_probe_version` | `{{ llama_version }}` | Version inspected by the opt-in `llama-probe` preflight. |

Override `llama_version`, `llama_artifact_url`, and `llama_artifact_checksum` together. The role intentionally leaves the artifact values empty: llama.app selects a CPU-feature-specific executable, and one host's binary is not necessarily runnable on another host.

## Artifact preflight

Run the opt-in preflight on the target host to inspect the same backend order as `llama.app/install.sh` and print copyable artifact settings:

```sh
ansible-playbook install-llama.yml --tags llama-probe
```

The preflight is tagged `never`, so a normal role run cannot trigger it. It only creates a temporary directory, downloads and executes version-pinned upstream probe helpers, downloads the suggested compressed executable to calculate its SHA-256, prints a YAML block, and removes the temporary directory. It does not create users, install `llama`, create units, or start services.

The helper executables are upstream native binaries. Run this preflight only when that trust boundary is acceptable. Review and store its output in inventory before normal provisioning.

## Instances

`llama_instances` contains one or more maps. Each map needs a unique `name`, `api_url`, and `mode` (`router` or `single_model`). It also has these fields:

| Field | Description |
| --- | --- |
| `environment` | Ordered `KEY=value` strings rendered as systemd `Environment=` entries. Use any official `LLAMA_ARG_*` variable or backend runtime variable. |
| `environmentfile` | Optional existing systemd `EnvironmentFile=` path. The role does not create it; use it for `HF_TOKEN`, `LLAMA_API_KEY`, and similar secrets. |
| `arguments` | Ordered raw `llama serve` arguments. Upstream command-line arguments override the corresponding environment variables. |
| `enabled` | Whether systemd enables the unit. |
| `started` | Whether systemd starts the unit. |

The default instance is router mode on `127.0.0.1:8080`. In router mode, omit `LLAMA_ARG_MODEL`; llama.cpp then provides dynamic model routing through its cache, models directory, or model presets. All instances share `llama_cache_directory` by default, so downloaded Hugging Face models are not duplicated.

Systemd does not expand `$HOME` in `Environment=` or `EnvironmentFile=` directives. Use absolute paths in values and file paths. The environment file must use systemd's `KEY=value` format and be readable by `llama_user`.

## Examples

```yaml
- name: Install a local GGUF server
  hosts: inference_hosts
  become: true
  roles:
    - role: githubixx.ai.llama
      vars:
        llama_version: "b10107"
        llama_artifact_url: "https://example.invalid/llama-app.zst"
        llama_artifact_checksum: "sha256:<verified-digest>"
        llama_instances:
          - name: llama-qwen
            api_url: "http://127.0.0.1:8080"
            mode: single_model
            environment:
              - "HOME={{ llama_home }}"
              - "LLAMA_CACHE={{ llama_cache_directory }}"
              - "LLAMA_ARG_HOST=127.0.0.1"
              - "LLAMA_ARG_PORT=8080"
              - "LLAMA_ARG_MODEL=/srv/models/Qwen3-8B-Q4_K_M.gguf"
              - "LLAMA_ARG_CTX_SIZE=8192"
              - "LLAMA_ARG_N_GPU_LAYERS=auto"
              - "LLAMA_ARG_N_PARALLEL=2"
            environmentfile: "/etc/llama/llama-qwen.env"
            arguments: []
            enabled: true
            started: true
```

Use Hugging Face model retrieval by replacing `LLAMA_ARG_MODEL` with an upstream repository setting:

```yaml
environment:
  - "HOME={{ llama_home }}"
  - "LLAMA_CACHE={{ llama_cache_directory }}"
  - "LLAMA_ARG_HOST=127.0.0.1"
  - "LLAMA_ARG_PORT=8080"
  - "LLAMA_ARG_HF_REPO=ggml-org/Qwen3.5-0.8B-GGUF:Q4_K_M"
```

For a router instance with local models or an INI preset, leave `LLAMA_ARG_MODEL` unset and use `LLAMA_ARG_MODELS_DIR` or `LLAMA_ARG_MODELS_PRESET`. See the [upstream server documentation](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) for the complete environment variable and router configuration surface.

## API exposure

The role checks loopback endpoints only. It uses `/health` for `single_model` instances and `/models` for `router` instances. Binding an instance to a non-loopback address exposes it without configuring TLS, authentication, a reverse proxy, or firewall policy. Configure those controls separately before exposing a server.

## Testing

The default Molecule scenario covers Ubuntu 24.04, Ubuntu 26.04, and Arch Linux. It passes an explicit test artifact through scenario variables, then verifies the binary, service account, shared cache, enabled service, and loopback router API.

See the [upstream maintenance guide](../../docs/llama-upstream.md) when reviewing a new llama.app release or changes to its installer.
