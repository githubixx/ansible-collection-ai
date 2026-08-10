# githubixx.ai.llamafile

Installs selected pre-built [llamafile](https://docs.mozilla.ai/llamafile/getting-started/pre-built-llamafiles) model bundles with SHA-256 verification. The role supports Ubuntu 24.04, Ubuntu 26.04, and Arch Linux on x86_64 with systemd.

No model is selected by default. This prevents an unexpected multi-gigabyte download.

## Requirements

- Ansible Core 2.15 or later.
- An AVX-capable x86_64 CPU. llamafile refuses to run without AVX.
- Network access to Hugging Face, or a configured model mirror.
- Enough free space for every selected final artifact and its in-progress partial download.

The role does not install GPU drivers, CUDA, ROCm/HIP, Vulkan, firewall rules, or model access credentials. Model licenses and their terms remain the operator's responsibility.

## Model selection

Set `llamafile_models` to the pre-built model IDs from `llamafile_catalog`. Every catalog entry uses an immutable Hugging Face revision URL and its LFS SHA-256 value. An entry may set `url`, `checksum`, and optionally `filename` to use a mirror.

```yaml
- name: Install selected llamafiles
  hosts: inference_hosts
  become: true
  roles:
    - role: githubixx.ai.llamafile
      vars:
        llamafile_models:
          - name: qwen-small
            id: qwen35-08b-q8-0
            run_mode: server
            host: 127.0.0.1
            port: 8080
            gpu: nvidia
            gpu_layers: 999
            context_size: 8192
            enabled: true
            started: true
          - name: ministral
            id: ministral-3-3b-q4-k-m
            run_mode: server
            host: 127.0.0.1
            port: 8081
            enabled: false
            started: false
          - name: aperture-chat
            id: apertus-8b
            run_mode: combined
```

This downloads all three bundles, starts only `qwen-small.service`, and installs `/usr/local/bin/aperture-chat-llamafile` for interactive use.

## Server mode

`run_mode: server` creates a systemd service called `<name>.service`. It runs the selected bundle with `--server`; `--jinja` defaults to enabled. Configure `host`, `port`, `context_size`, `threads`, `gpu_layers`, `gpu`, `arguments`, `environment`, and `environmentfile` per entry. Multiple server services are allowed, but the role does not estimate GPU/RAM capacity; use distinct addresses and ports.

The default bind address is `127.0.0.1`. Binding a non-loopback address exposes an unauthenticated HTTP service. Configure TLS, authentication, and firewall policy outside this role.

The role waits for `GET /health` only for started services bound to `127.0.0.1`, `localhost`, or `::1`. Override `llamafile_healthcheck_path` if upstream changes the endpoint.

## Combined mode

Upstream combined mode is the default when none of `--cli`, `--chat`, or `--server` is passed: it starts an HTTP server and an interactive terminal chat together. A systemd unit has no operator terminal, so combined entries intentionally cannot be enabled or started by the role.

For an entry with `run_mode: combined`, invoke its launcher from a terminal:

```sh
aperture-chat-llamafile
```

The launcher preserves standard input/output, applies the selected model's GPU and runtime options, and accepts further llamafile arguments.

## Large downloads and retries

Artifacts are root-owned under `/opt/llamafile/models`. Each transfer runs asynchronously, uses `curl --continue-at -`, follows redirects, retries transient failures, verifies SHA-256, and installs atomically only after verification.

- Partial files: `/var/lib/llamafile/downloads/<name>.part`
- Transfer logs: `/var/lib/llamafile/download-logs/<name>.log`
- Default controller deadline: `llamafile_download_async_timeout: 86400`
- Retry controls: `llamafile_download_retries`, `llamafile_download_retry_delay`, and `llamafile_download_connect_timeout`
- Poll control: `llamafile_download_poll_interval`

Failed partial downloads and logs are retained by default; rerunning the play resumes the partial file. Set `llamafile_download_cleanup_partial: true` only when a fresh transfer is required.

## GPU support

On Linux, NVIDIA acceleration requires the CUDA SDK with `nvcc` on the service account's `PATH`; AMD acceleration requires the HIP SDK with usable `hipcc`. Use `gpu: nvidia`, `amd`, or `vulkan` to turn a missing backend into a startup failure rather than the upstream silent CPU fallback. `gpu: disable` forces CPU inference; `gpu: auto` leaves backend selection to llamafile. NVIDIA and AMD deployments normally set `gpu_layers: 999`.

See the [upstream GPU support reference](https://docs.mozilla.ai/llamafile/reference/support) for supported hardware and backend limitations.

See the [upstream maintenance guide](../../docs/llamafile-upstream.md) when reviewing changes to the pre-built catalog or llamafile 0.10.*.
