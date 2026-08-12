# githubixx.ai.comfyui

Installs ComfyUI from a pinned upstream Git tag in a dedicated Python virtual environment and runs it as a systemd service. The role follows ComfyUI's manual installation process: install a selected PyTorch build, then install the ComfyUI requirements. It does not install, configure, or validate graphics drivers, CUDA, ROCm, NPU firmware, or vendor toolkits.

## Requirements

- Ansible Core 2.15 or later.
- Linux x86_64 running Ubuntu 24.04, Ubuntu 26.04, or Arch Linux.
- systemd.
- Network access to GitHub, PyPI, and the selected PyTorch package index during installation.
- An already-installed and compatible driver/runtime for every non-CPU accelerator profile.

The default `cpu` profile works without an accelerator and starts ComfyUI with `--cpu`. It is useful for testing and low-throughput workloads.

## Role variables

All public variables use the `comfyui_` prefix.

| Variable | Default | Description |
| --- | --- | --- |
| `comfyui_version` | `v0.32.0` | Pinned upstream ComfyUI Git tag. |
| `comfyui_repository` | ComfyUI upstream repository | Git repository to clone. |
| `comfyui_accelerator` | `cpu` | Explicit PyTorch resource profile. |
| `comfyui_pytorch_packages` | Profile packages | PyTorch packages to install into the virtual environment. Override for the `external` profile. |
| `comfyui_pytorch_pip_args` | Profile index arguments | Additional pip arguments for the selected PyTorch package source. |
| `comfyui_manager_enabled` | `true` | Install upstream Manager requirements and launch with `--enable-manager`. |
| `comfyui_custom_nodes` | `[]` | Pinned third-party custom node repositories. |
| `comfyui_install_directory` | `/opt/comfyui` | ComfyUI checkout and virtual environment directory. |
| `comfyui_home` | `/var/lib/comfyui` | Dedicated service account home, cache, and configuration directory. |
| `comfyui_user` | `comfyui` | Dedicated system service account. |
| `comfyui_host` | `127.0.0.1` | ComfyUI bind address. |
| `comfyui_port` | `8188` | ComfyUI HTTP port. |
| `comfyui_service_environment` | ComfyUI service environment | Ordered systemd environment entries, each written as an `Environment=` directive. |
| `comfyui_service_environmentfile` | Empty | Optional systemd environment file. |
| `comfyui_service_arguments` | Listen address and port | Additional ComfyUI command-line arguments. |
| `comfyui_service_enabled` | `true` | Enable the system service. |
| `comfyui_service_started` | `true` | Start the system service. |

## Accelerator profiles

Set `comfyui_accelerator` explicitly; the role never probes PCI devices or chooses an accelerator automatically.

| Profile | PyTorch source | Prerequisite outside this role |
| --- | --- | --- |
| `cpu` | PyPI | None. |
| `nvidia_cuda` | PyTorch CUDA 13.0 index | Compatible NVIDIA driver. |
| `amd_rocm` | PyTorch ROCm 7.2 index | Compatible AMD ROCm driver/runtime. |
| `amd_rdna3` | AMD RDNA 3 nightly index | Compatible experimental AMD runtime. |
| `amd_rdna35` | AMD RDNA 3.5 nightly index | Compatible experimental AMD runtime. |
| `amd_rdna4` | AMD RDNA 4 nightly index | Compatible experimental AMD runtime. |
| `intel_xpu` | PyTorch Intel XPU index | Compatible Intel GPU driver/runtime. |
| `external` | Operator supplied | Vendor-specific PyTorch extension and toolkit. |

The `external` profile supports Ascend NPUs, Cambricon MLUs, Iluvatar Corex, and similar vendor runtimes after the operator has installed the vendor prerequisites. Set `comfyui_pytorch_packages` and, if needed, `comfyui_pytorch_pip_args` to the vendor-supported values. The role rejects an `external` profile with no packages configured.

## Example

```yaml
- name: Install ComfyUI with NVIDIA acceleration
  hosts: inference_hosts
  become: true
  roles:
    - role: githubixx.ai.comfyui
      vars:
        comfyui_accelerator: nvidia_cuda
```

The role manages `comfyui.service` as the dedicated `comfyui` account. The application, virtual environment, and custom nodes are under `/opt/comfyui`; models, inputs, outputs, temporary files, user data, service configuration, and cache files are under `/var/lib/comfyui`.

## ComfyUI Manager and extensions

Current ComfyUI releases integrate ComfyUI Manager v4. With `comfyui_manager_enabled: true`, the role installs the upstream `manager_requirements.txt` and adds `--enable-manager` to the service command. It does not clone a Manager repository into `custom_nodes`.

Use `comfyui_custom_nodes` for third-party extensions. Pin each extension to a Git tag or commit and declare a requirements file only when the extension provides one:

```yaml
comfyui_custom_nodes:
  - name: example-node
    repo: https://github.com/example/ComfyUI-Example-Node.git
    version: "1.2.3"
    requirements_file: requirements.txt
```

Custom node code runs inside the ComfyUI service process. Review each extension and pin immutable revisions before deploying it. Extension-specific install scripts, model downloads, and post-install commands are outside this role's scope.

## Localhost example

Create `install-comfyui.yml`:

```yaml
---
- name: Install ComfyUI locally
  hosts: localhost
  connection: local
  become: true
  collections:
    - githubixx.ai
  roles:
    - role: comfyui
      vars:
        comfyui_accelerator: cpu
```

Run it with:

```sh
ansible-playbook install-comfyui.yml --ask-become-pass
```

## API exposure

The default bind address is loopback-only. Changing `comfyui_host` to a non-loopback address exposes ComfyUI without configuring authentication, TLS, or firewall rules. Configure those controls separately before exposing the service.

## Testing

The default Molecule scenario uses Vagrant with libvirt and verifies Ubuntu 24.04, Ubuntu 26.04, and Arch Linux with the `cpu` profile. It validates the pinned checkout, virtual environment, Manager enablement, systemd service, and loopback `/system_stats` endpoint. It intentionally does not download a model or execute a workflow.

GPU, NPU, and MLU profiles need acceptance testing on representative hardware with the desired vendor runtime and model workflow.
