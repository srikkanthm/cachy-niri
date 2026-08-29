# NVIDIA Driver Setup on CachyOS (reference)

**Date:** 2026-08-24
**Purpose:** Reference for replicating this driver setup on another distro.

## Hardware

- **GPU:** NVIDIA GeForce 930M (GM108M, Maxwell), 2048 MiB, PCIe `01:00.0`
- Hybrid laptop: Intel HD Graphics 520 (Skylake) is the primary/display GPU (`i915`); NVIDIA is the secondary render GPU (`3D controller`)
- Display server: Wayland (niri compositor + Noctalia)
- Kernel driver in use: `nvidia` (modules loaded: `nvidia`, `nvidia_drm`, `nvidia_modeset`, `nvidia_uvm`)

## Key insight: why the 580 legacy branch

The 930M is a Maxwell card. Current upstream Arch `nvidia` packages target newer GPUs / open kernel modules, so this system uses the **legacy 580 series proprietary driver from the AUR**:

| Package | Version | Notes |
|---|---|---|
| `nvidia-580xx-dkms` | 580.178.04-1 | Kernel module, built with DKMS |
| `nvidia-580xx-utils` | 580.178.04-1 | Userspace libs (GLX/EGL/Vulkan) |
| `lib32-nvidia-580xx-utils` | 580.178.04-1 | 32-bit (Steam/Wine) |
| `opencl-nvidia-580xx` | 580.178.04-1 | OpenCL |
| `lib32-opencl-nvidia-580xx` | 580.178.04-1 | 32-bit OpenCL |
| `nvidia-580xx-settings` | 580.159.03-1 | nvidia-settings GUI |
| `libva-nvidia-driver` | 0.0.17-1 | VA-API bridge (NVDEC video accel) |
| `linux-firmware-nvidia` | 1:20260810-2 | Firmware |

## Supporting graphics stack

| Package | Version |
|---|---|
| `mesa` / `lib32-mesa` | 26.2.1 (Intel side) |
| `vulkan-intel` / `lib32-vulkan-intel` | 26.2.1 |
| `vulkan-icd-loader` / `lib32-vulkan-icd-loader` | 1.4.357.0 |
| `egl-wayland` | 1.1.21 |
| `egl-gbm` | 1.1.3 |
| `egl-x11` | 1.0.5 |
| `eglexternalplatform` | 1.2.1 |
| `libva` | 2.24.1 |

## Kernel / DKMS status

- Running kernel: `7.2.0-1-cachyos`
- DKMS module built for **two kernels**:
  - `nvidia/580.178.04 → 7.2.0-1-cachyos` (installed)
  - `nvidia/580.178.04 → 6.18.42-1-cachyos-lts` (installed)

## Configuration files

- `/etc/modprobe.d/` — **empty**, no custom NVIDIA options
- `/etc/X11/xorg.conf.d/` — keyboard only, no xorg GPU config
- No `prime-run`/offload script installed; Intel is the default GLX/EGL renderer, NVIDIA available via EGL (`eglinfo` reports NVIDIA vendor)
- Pure Wayland setup — no Xorg driver config needed

## Verification commands used

```bash
lspci -k | grep -A3 -iE 'vga|3d'        # GPU + kernel driver in use
dkms status                              # module builds per kernel
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv   # CUDA 13.0
glxinfo -B                               # primary renderer (Intel/Mesa)
eglinfo                                  # EGL providers incl. NVIDIA
```

## Notes for porting to another distro

1. Install the **proprietary legacy 580 branch** (e.g. distro equivalent of `nvidia-580xx-dkms`; on Ubuntu it's `nvidia-driver-580`, on Fedora use RPM Fusion's 580xx akmod)
2. Ensure the module is built for **every installed kernel** (DKMS handles this)
3. On Wayland, `nvidia_drm.modeset=1` should be enabled (default since driver 560)
4. Install `egl-wayland` equivalents for compositor support
5. For hybrid graphics keep Mesa/i915 as the primary; NVIDIA renders offload workloads only
