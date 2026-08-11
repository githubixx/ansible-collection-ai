# githubixx.ai

Ansible collection for AI-related software.

## Roles

- `githubixx.ai.comfyui` installs ComfyUI from a pinned upstream Git tag in a dedicated Python virtual environment and runs it as a local systemd service.
- `githubixx.ai.llama` installs llama.cpp's `llama serve` binary from a pinned, SHA-256-verified artifact and manages one or more systemd services.
- `githubixx.ai.llamafile` installs selected pre-built llamafile model bundles from a pinned upstream catalog, with server and interactive combined modes.
- `githubixx.ai.openshell` installs the NVIDIA OpenShell CLI and local gateway from pinned release archives.
- `githubixx.ai.ollama` installs Ollama as a local system service from pinned release archives.
- `githubixx.ai.openwebui` installs Open WebUI from a pinned PyPI package as a local systemd service.
- `githubixx.ai.vllm` installs vLLM from a pinned PyPI package as a local OpenAI-compatible systemd service.

For requirements and usage see:

- [ComfyUI role documentation](roles/comfyui/README.md)
- [llama role documentation](roles/llama/README.md)
- [llamafile role documentation](roles/llamafile/README.md)
- [OpenShell role documentation](roles/openshell/README.md)
- [Ollama role documentation](roles/ollama/README.md)
- [Open WebUI role documentation](roles/openwebui/README.md)
- [vLLM role documentation](roles/vllm/README.md)

## Development

Build the collection from this directory with `ansible-galaxy collection build`. The collection declares Ansible Core 2.15 or later in [meta/runtime.yml](meta/runtime.yml).

GitHub Release notes are generated from Conventional Commit subjects. Use `feat:` for additions and `fix:` for bug fixes. Use `refactor:`, `perf:`, `build:`, `ci:`, `chore:`, or `deps:` for changes; other commit types are omitted from generated release notes.
