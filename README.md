# Hybrid GPU (Intel + NVIDIA) Wayland Sleep & Idle Freeze

[![Linux Kernel](https://img.shields.io/badge/Kernel-6.x%20%7C%207.x-blue.svg)](https://kernel.org)
[![Desktop](https://img.shields.io/badge/Compositor-GNOME%20Mutter%20%7C%20Wayland-orange.svg)](https://gitlab.gnome.org/GNOME/mutter)
[![NVIDIA Driver](https://img.shields.io/badge/NVIDIA-550%2B%20%7C%20560%2B%20%7C%20610%2B-green.svg)](https://github.com/NVIDIA/open-gpu-kernel-modules)
[![Distros](https://img.shields.io/badge/Distros-Fedora%20%7C%20Ubuntu%20%7C%20Arch%20%7C%20openSUSE-purple.svg)](#affected-ecosystem--distributions)

> **A coordinated investigation and upstream contribution initiative** to resolve the display server deadlocks affecting dual-GPU Linux laptops (Intel/AMD iGPU + NVIDIA dGPU) connected to external monitors over USB-C DisplayPort Alternate Mode under Wayland.

---

## The Problem: "The 5-Minute Display Lockup"

Every day, thousands of Linux laptop users experience a frustrating, seemingly random graphical freeze:

1. **The Idle Lockup**: You step away from your desk. After 5 minutes of inactivity, your screens blank for power-saving (DPMS off). When you return and touch the mouse or keyboard, your laptop screen remains black, your external USB-C monitor is permanently frozen showing a blurred lock screen wallpaper, and all mouse/keyboard inputs are completely ignored.
2. **The Sleep Bounce**: You put your laptop into suspend and place it in your bag. Within 5 seconds, an external peripheral or background timer bounces the system awake. When you open the lid hours later, both screens are dead black, the laptop is burning hot, and the battery is drained.

### Why This Has Baffled Users for Years
* **The OS isn't crashed**: Background Linux operations (SSH daemon, network connections, containers, audio playback, cron tasks) continue running perfectly.
* **Only the graphical display server and input processing are deadlocked**.
* Because the failure is silent, users are forced to perform a hard, ungraceful power button shutdown, losing unsaved work.
* Because the bug crosses **four distinct open-source boundaries** (Distro Packagers, Hardware Vendor Kernel Modules, Wayland Compositors, and Linux Power Management), bug reports have historically been bounced between project trackers without resolution.

**This project exists to unify the forensic evidence, provide reliable 30-second reproductions, and coordinate targeted upstream fixes across all affected layers.**

---

## Upstream Coordination Dashboard

| Upstream Project | Component | Tracking Link | Objective | Status |
| :--- | :--- | :--- | :--- | :---: |
| **RPM Fusion / Fedora** | `xorg-x11-drv-nvidia` | *Pending submission* | Condition `nvidia-suspend-nofreeze.conf` on legacy X11; configure S0ix/VRAM module defaults | **Drafted** |
| **NVIDIA Open Kernel** | `nvidia-drm` / `nvidia.ko` | *Pending submission* | Fallback to D3hot on active DP Alt Mode connectors; auto-enable S0ix on `s2idle`-only platforms | **Drafted** |
| **GNOME Mutter** | `meta-kms-impl-device-atomic.c` | *Pending submission* | Gracefully handle `EPERM` / `EACCES` during DRM Master loss without poisoning KMS state | **Drafted** |
| **Desktop Settings** | GNOME Settings / COSMIC | *Feature proposal* | Provide user-facing toggle to configure peripheral wake policy (keyboard wake vs. mouse wake) | **Planned** |

---

## Affected Ecosystem & Distributions

This is **not** an isolated hardware fault or single-distribution issue:

### 1. Distributions Affected
* **Fedora Linux (39 through 44+)**: Directly affected by RPM Fusion's unconditional `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false` drop-in and commented-out power management defaults.
* **Arch Linux / EndeavourOS**: Affected by NVIDIA driver D3cold power cutoff and missing S0ix parameter configurations.
* **Ubuntu & Debian**: Systems with NVIDIA proprietary/open drivers using Wayland and USB-C docks exhibit the exact same D3cold modeset hangs and Modern Standby corruptions.
* **openSUSE (Tumbleweed / Leap)**: Subject to the same kernel module runtime power management deadlocks.

### 2. Known Issues & Forum Threads Solved by This Initiative
By resolving the root causes documented here, several long-standing Linux graphics bugs are resolved simultaneously:
* **The "External display stuck on blurred lock screen" bug**: Directly caused by NVIDIA entering D3cold during DPMS off and failing atomic link-training upon re-wake.
* **The "Black screen after resume on modern laptops" bug**: Caused by platforms that only support ACPI `s2idle` (Modern Standby) attempting sleep without `NVreg_EnableS0ixPowerManagement=1`.
* **The "Laptop burning hot in backpack" syndrome**: Caused by S0ix suspend failures and immediate resume bounce cycles.
* **`drmModeAtomicCommit: Permission denied` log flood**: Caused by Wayland compositors remaining unfrozen during systemd suspend.

### 3. Peripheral Hardware Scope
* **Hardware-Agnostic**: The graphics deadlock is triggered by *any* wakeup source (Logitech Bolt, Logitech Unifying, Razer, Corsair, Dell USB docks, Bluetooth radios, or incoming network wake packets).
* **Wakeup Policy**: While optical mouse sensor vibrations can trigger premature wakeups, the ability to wake a suspended laptop by tapping a wireless keyboard is an essential user feature. Wakeup policy should be handled by desktop environment settings, not by graphics hacks or global udev bans.

---

## Hardware Topology & Architecture

Tested and validated on hybrid architectures where the internal display is driven by the integrated GPU and external USB-C ports are wired directly to the NVIDIA discrete GPU:

```
       +----------------------------------------------------------------+
       |                          GNOME Shell                           |
       |                (Mutter Wayland Display Server)                 |
       +-------------------------------+--------------------------------+
                                       |
                    Atomic Modesetting (libdrm / KMS)
                                       |
                  +---------------------+---------------------+
                  |                                           |
                  v                                           v
        +-------------------+                       +-------------------+
        |    card1 (i915)   |                       | card0 (nvidia-drm)|
        |   Intel ArrowLake |                       |  NVIDIA Blackwell |
        +---------+---------+                       +---------+---------+
                  |                                           |
            (eDP-1 Internal)                            (DP-5 USB-C Alt)
                  |                                           |
                  v                                           v
        +-------------------+                       +-------------------+
        |   Laptop Screen   |                       |  External Monitor |
        +-------------------+                       +-------------------+
```

### Reference Test System
* **Host**: HP Laptop (Intel Arrow Lake-S / Meteor Lake architecture)
* **iGPU**: Intel Corporation Arrow Lake-S Graphics (`[8086:7d67]`, driver: `i915`, `/dev/dri/card1`) -> Internal Panel `eDP-1`
* **dGPU**: NVIDIA Corporation GB207GLM [RTX PRO 1000 Blackwell Laptop GPU] (`[10de:2db8]`, driver: `nvidia-drm` 610.57.04, `/dev/dri/card0`) -> External USB-C `DP-5`
* **OS / Desktop**: Fedora Linux 44, Linux Kernel 7.2.4, GNOME Shell 50.4 on Wayland
* **ACPI Sleep Capabilities**: `cat /sys/power/mem_sleep` -> **`[s2idle]`** (S0ix Modern Standby only; legacy S3 is unsupported in hardware).

---

## Root Cause Analysis: The Three Critical Layers

```
                                SYSTEM SUSPEND / DPMS EVENT
                                             │
             ┌───────────────────────────────┴───────────────────────────────┐
             ▼                                                               ▼
  [Layer 1: Distro Packaging]                                    [Layer 2: NVIDIA Kernel]
  SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false                       NVreg_DynamicPowerManagement=3
  leaves user session unfrozen.                                  powers down GPU to D3cold on DPMS.
             │                                                               │
             ▼                                                               ▼
  systemd-logind revokes DRM Master                              Mutter issues atomic commit
  while Mutter is still active.                                  over USB-C DP Alt Mode.
             │                                                               │
             ▼                                                               ▼
  drmModeAtomicCommit returns EPERM                              nvidia-drm blocks indefinitely
             │                                                   waiting on powered-down silicon.
             └───────────────────────────────┬───────────────────────────────┘
                                             │
                                             ▼
                                 [Layer 3: GNOME Mutter]
                                 KMS worker thread stalls;
                                 Main thread blocks synchronously;
                                 libinput processing halts.
                                             │
                                             ▼
                                 TOTAL SYSTEM UI FREEZE
```

### Layer 1: Distro Packaging (RPM Fusion / Fedora)
* **The Culprit**: `/usr/lib/systemd/system/systemd-suspend.service.d/nvidia-suspend-nofreeze.conf` sets `Environment = SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false`.
* **The Why**: Historically introduced for legacy Xorg when the closed driver required a virtual terminal switch (`chvt 63`) on suspend. Freezing Xorg in `user.slice` prevented it from handling the VT switch.
* **The Bug Under Wayland**: Wayland sessions perform no VT switch. With `UseKernelSuspendNotifiers: 1`, NVIDIA's kernel module handles suspend directly via PM notifiers. However, leaving `user.slice` unfrozen allows GNOME Shell to continue issuing KMS atomic commits concurrently as `systemd-logind` drops DRM Master, throwing `drmModeAtomicCommit: Permission denied`.
* **Missing Defaults**: Modprobe defaults leave `NVreg_EnableS0ixPowerManagement=0` and `NVreg_PreserveVideoMemoryAllocations=0`, corrupting GPU memory upon waking from modern standby.

### Layer 2: NVIDIA Open Kernel Modules (`nvidia-drm` / `nvidia.ko`)
* **The Culprit**: `NVreg_DynamicPowerManagement` defaults to `0x03` (fine-grained runtime D3cold).
* **The Bug**: When an external monitor is driven over USB-C DisplayPort Alt Mode and enters DPMS sleep, the GPU silicon is powered down into D3cold. When Mutter subsequently sends KMS commits to adjust CRTC states or wake the monitor, `nvidia-drm` fails to properly power-gate the GPU before attempting DisplayPort AUX communication, deadlocking the kernel worker.
* **Missing S0ix Logic**: The driver does not auto-detect systems where the BIOS offers *only* `s2idle`, failing to coordinate modern standby.

### Layer 3: GNOME Mutter (Wayland Compositor)
* **The Culprit**: In `meta-kms-impl-device-atomic.c`, `PowerSaveMode` transitions synchronously wait for the KMS thread.
* **The Bug**: When `drmModeAtomicCommit` returns `-EPERM` during DRM Master loss, Mutter's atomic state machine treats the error as fatal and leaves pending in-flight updates poisoned. The KMS thread enters an unrecoverable state, stalling Mutter's main event loop and freezing `libinput` keyboard/mouse handling.

---

## 30-Second Reliable Reproduction

You can reliably trigger and confirm the DPMS freeze on an affected system without waiting for idle timeouts:

```bash
# 1. Open an SSH terminal or secondary shell to monitor logs:
journalctl -f | grep -E "PowerSaveMode|gnome-shell|nvidia"

# 2. In your desktop terminal, issue a direct DPMS turn-off command:
busctl --user set-property org.gnome.Mutter.DisplayConfig /org/gnome/Mutter/DisplayConfig org.gnome.Mutter.DisplayConfig PowerSaveMode i 1
```

* **Observed Result**: The internal panel goes dark; the external USB-C monitor permanently freezes displaying the current desktop/lock screen frame; mouse and keyboard stop responding; after ~25–40 seconds, the journal reports:
  ```text
  gsd-power: Error setting property 'PowerSaveMode' on interface org.gnome.Mutter.DisplayConfig: Timeout was reached (g-io-error-quark, 24)
  at-spi2-registryd: Disabling unresponsive app with pid [gnome-shell-pid]
  ```

---

## Immediate Local Workarounds (For End Users)

If you are experiencing these freezes on your machine today, run this script to apply local client-side mitigations while upstream patches are processed:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==> 1. Configuring NVIDIA S0ix, D3hot & Video Memory Preservation..."
sudo tee /etc/modprobe.d/nvidia-power-management.conf << 'EOF'
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_TemporaryFilePath=/var/tmp
options nvidia NVreg_EnableS0ixPowerManagement=1
options nvidia NVreg_DynamicPowerManagement=0x01
EOF

echo "==> 2. Enabling NVIDIA Systemd Sleep Services..."
sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service

echo "==> 3. Masking obsolete unfreeze drop-ins for Wayland..."
for service in systemd-suspend systemd-hibernate systemd-hybrid-sleep systemd-suspend-then-hibernate; do
    sudo mkdir -p "/etc/systemd/system/${service}.service.d"
    sudo ln -sf /dev/null "/etc/systemd/system/${service}.service.d/nvidia-suspend-nofreeze.conf"
done
sudo systemctl daemon-reload

echo "==> 4. Disabling automatic suspend on AC power in GNOME..."
if [ -n "${SUDO_USER:-}" ]; then
    sudo -u "$SUDO_USER" dbus-launch gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing' || true
else
    gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing' || true
fi

echo "==> 5. Rebuilding initramfs..."
sudo dracut -f

echo "==> Mitigations applied! Please reboot your system."
```

### Verification After Reboot
Verify that the parameters are active in your running kernel:
```bash
cat /proc/driver/nvidia/params | grep -E "EnableS0ixPowerManagement|PreserveVideoMemoryAllocations|DynamicPowerManagement"
```
* **Expected Output**:
  ```text
  PreserveVideoMemoryAllocations: 1
  EnableS0ixPowerManagement: 1
  DynamicPowerManagement: 1
  ```

---

## How to Contribute & Join the Discussion

We are actively seeking feedback, additional forensic logs, and test results from different laptop vendors:

1. **Test Your System**: Run the 30-second reproduction command above and let us know if your system freezes or recovers cleanly.
2. **Share Hardware Reports in [GitHub Discussions](https://github.com/x3m-industries/hybrid-gpu-wayland-freeze/discussions)**: Include your output from:
   ```bash
   inxi -Gz
   cat /sys/power/mem_sleep
   cat /proc/driver/nvidia/params | grep -i dynamic
   ```
3. **Upstream PR Testing**: Once upstream pull requests are opened, we will link them in the [Upstream Coordination Dashboard](#upstream-coordination-dashboard) so community members can test and add review tags.
