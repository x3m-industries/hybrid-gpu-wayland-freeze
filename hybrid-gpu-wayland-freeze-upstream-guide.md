# Complete Engineering & Contribution Guide: Hybrid GPU (Intel + NVIDIA) Wayland Sleep/Idle Freezes

This document provides an end-to-end technical breakdown of the display server deadlock that occurs on hybrid-GPU Linux laptops (Intel iGPU + NVIDIA dGPU) with external USB-C displays under Wayland. It covers the root causes across four distinct open-source layers, step-by-step reproduction instructions, validation and testing procedures, and ready-to-submit contribution proposals for upstream projects.

---

## Table of Contents
1. [Hardware Topology & Environment](#1-hardware-topology--environment)
2. [Symptoms & Failure Modes](#2-symptoms--failure-modes)
3. [Forensic Log Analysis](#3-forensic-log-analysis)
4. [In-Depth Root Cause Analysis](#4-in-depth-root-cause-analysis)
   - [Layer 1: Distro Packaging (RPM Fusion / Fedora)](#layer-1-distro-packaging-rpm-fusion--fedora)
   - [Layer 2: NVIDIA Open Kernel Modules](#layer-2-nvidia-open-kernel-modules)
   - [Layer 3: GNOME Mutter (Wayland Compositor)](#layer-3-gnome-mutter-wayland-compositor)
   - [Layer 4: USB & Peripheral Power Management](#layer-4-usb--peripheral-power-management)
5. [Reliable Reproduction Procedure](#5-reliable-reproduction-procedure)
6. [Testing & Verification Matrix](#6-testing--verification-matrix)
7. [Immediate Local Workarounds & Automation Script](#7-immediate-local-workarounds--automation-script)
   - [Automated Re-Apply Script](#automated-re-apply-script-one-liner)
   - [Step-by-Step Manual Instructions](#step-by-step-manual-instructions)
   - [Diagnostic Commands for Future Freezes](#diagnostic-commands-for-future-freezes)
8. [Upstream Contribution Roadmap & Patch Templates](#8-upstream-contribution-roadmap--patch-templates)
   - [Contribution 1: RPM Fusion Packaging Fix](#contribution-1-rpm-fusion-packaging-fix-highest-impact)
   - [Contribution 2: NVIDIA Open Kernel Driver Fix](#contribution-2-nvidia-open-kernel-driver-fix)
   - [Contribution 3: GNOME Mutter Resilience Fix](#contribution-3-gnome-mutter-resilience-fix)
   - [Contribution 4: Systemd / Udev HWDB Rule](#contribution-4-systemd--udev-hwdb-rule)

---

## 1. Hardware Topology & Environment

### Tested System Configuration
* **Laptop Platform**: HP Laptop (Intel Arrow Lake-S architecture)
* **Integrated GPU (iGPU)**: Intel Corporation Arrow Lake-S Graphics (`[8086:7d67]`, rev 06)
  * Kernel Driver: `i915`
  * DRM Node: `/dev/dri/card1`
  * Connected Display: Internal Panel `eDP-1` (1920x1200 @ 60Hz, Scale 1.0)
* **Discrete GPU (dGPU)**: NVIDIA Corporation GB207GLM [RTX PRO 1000 Blackwell Generation Laptop GPU] (`[10de:2db8]`, rev a1)
  * Kernel Driver: `nvidia` / `nvidia-drm` (Open Kernel Module 610.57.04)
  * DRM Node: `/dev/dri/card0`
  * Connected Display: External Monitor via USB-C DisplayPort Alt Mode `DP-5` (2160x1440 @ 60Hz, Scale 1.5)
* **Operating System**: Fedora Linux 44 (Workstation Edition)
* **Kernel Version**: Linux `7.2.4-200.fc44.x86_64`
* **Desktop Environment**: GNOME Shell 50.4 on Wayland (`mutter-50.4`)
* **ACPI Sleep Capabilities**: `cat /sys/power/mem_sleep` reports **`[s2idle]`** only (Modern Standby / S0ix). Traditional ACPI S3 (Deep Sleep) is unsupported by hardware firmware.

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

---

## 2. Symptoms & Failure Modes

When left idle or transitioning through sleep cycles, the machine enters a total graphical freeze. Two distinct manifestations occur:

### Manifestation A: Idle DPMS Screen Blanking Deadlock
1. The user steps away. After the idle delay (default: 5 minutes), GNOME Shell locks the screen and invokes display power saving (DPMS off).
2. The internal laptop screen turns black (Intel backlight off).
3. The external USB-C screen stays frozen, permanently showing the blurred lock screen background.
4. GNOME Shell stops responding to mouse clicks, keyboard input, and the physical power button short press.
5. All background operating system tasks (kernel networking, cron, background containers, SSH) continue running normally.

### Manifestation B: Sleep Bounce Deadlock
1. After 15 minutes of idle on AC power, GNOME Shell initiates system suspend (`s2idle`).
2. A wireless peripheral (e.g. Logitech wireless receiver) or network card immediately wakes the system within 5–7 seconds.
3. Upon waking, GNOME Shell attempts to resume display management, but the NVIDIA GPU runtime power management is in an uncoordinated state.
4. Both screens remain completely black.
5. Hours later (e.g. morning), touching the keyboard generates input events in peripheral daemons (e.g. `solaar`), but GNOME Shell was already deadlocked since the resume event hours prior and cannot process input or re-enable displays.

---

## 3. Forensic Log Analysis

Historical logs across multiple system boots reveal an identical failure signature in `journalctl`:

### 1. Mutter KMS Deadlock during PowerSaveMode
```text
Sep 11 17:59:46 fedora gsd-power[4113]: Error setting property 'PowerSaveMode' on interface org.gnome.Mutter.DisplayConfig: Timeout was reached (g-io-error-quark, 24)
Sep 11 17:59:46 fedora gsd-power[4113]: couldn't set the auto brightness target: Timeout was reached
Sep 11 17:59:56 fedora at-spi2-registryd[4051]: Disabling unresponsive app with pid 3979
```
*Note: PID 3979 is `gnome-shell`. The D-Bus call timed out after 40 seconds because Mutter's main thread was blocked waiting on its KMS thread.*

### 2. DRM Master Revocation during Suspend
```text
Sep 11 17:58:45 fedora systemd-logind[1045]: The system will suspend now!
Sep 11 17:58:45 fedora systemd-sleep[34593]: User sessions remain unfrozen on explicit request ($SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=0).
Sep 11 17:58:45 fedora systemd-sleep[34593]: This is not recommended, and might result in unexpected behavior...
Sep 11 17:58:45 fedora kernel: PM: suspend entry (s2idle)
Sep 11 17:58:51 fedora kernel: PM: suspend exit
Sep 11 17:58:51 fedora gnome-shell[3979]: [atomic] Failed to disable device '/dev/dri/card1': drmModeAtomicCommit: Permission denied
Sep 11 17:58:51 fedora gnome-shell[3979]: [atomic] Failed to disable device '/dev/dri/card0': drmModeAtomicCommit: Permission denied
```
*Note: GNOME Shell was kept unfrozen by systemd. When logind took over DRM Master to initiate sleep, GNOME Shell's delayed atomic commit failed with `EPERM` / `Permission denied`, breaking Mutter's internal atomic state machine.*

### 3. Immediate Resume Bounce & PCI Quirk Trigger
```text
Sep 11 23:47:27 fedora kernel: PM: suspend entry (s2idle)
Sep 11 23:47:27 fedora kernel: nvidia 0000:02:00.0: Enabling HDA controller
Sep 11 23:47:32 fedora kernel: igc 0000:81:00.0 enp129s0: Timeout reading IGC_PTM_STAT register
Sep 11 23:47:34 fedora kernel: PM: suspend exit
Sep 11 23:47:34 fedora gnome-shell[4023]: g_settings_get_value: assertion 'G_IS_SETTINGS (settings)' failed
Sep 11 23:48:29 fedora gsd-power[4147]: Error setting property 'PowerSaveMode' on interface org.gnome.Mutter.DisplayConfig: Timeout was reached (g-io-error-quark, 24)
Sep 11 23:48:36 fedora at-spi2-registryd[4082]: Disabling unresponsive app with pid 4023
```
*Note: Suspend lasted only 7 seconds. Right after resuming, `gsd-power` attempted DPMS, and Mutter deadlocked in `nvidia-drm`.*

---

## 4. In-Depth Root Cause Analysis

### Layer 1: Distro Packaging (RPM Fusion / Fedora)
* **Package**: `xorg-x11-drv-nvidia-power` (built from `xorg-x11-drv-nvidia.spec`)
* **Problem 1 (`nvidia-suspend-nofreeze.conf`)**:
  RPM Fusion installs four systemd drop-ins into:
  `/usr/lib/systemd/system/systemd-suspend.service.d/nvidia-suspend-nofreeze.conf`
  `/usr/lib/systemd/system/systemd-hibernate.service.d/nvidia-suspend-nofreeze.conf`
  `/usr/lib/systemd/system/systemd-hybrid-sleep.service.d/nvidia-suspend-nofreeze.conf`
  `/usr/lib/systemd/system/systemd-suspend-then-hibernate.service.d/nvidia-suspend-nofreeze.conf`
  
  These drop-ins set `Environment = SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false`.
  This was originally introduced as a workaround for Xorg when the proprietary driver requested VT switching on suspend. In Wayland, no VT switch is performed; instead, leaving user sessions unfrozen allows GNOME Shell to race against systemd-logind's DRM Master revocation, throwing `drmModeAtomicCommit: Permission denied` and corrupting Mutter's display state.
* **Problem 2 (Unconfigured Module Defaults)**:
  RPM Fusion ships `/usr/lib/modprobe.d/nvidia-power-management.conf` with all power-saving options commented out by default. S0ix modern standby is disabled (`NVreg_EnableS0ixPowerManagement=0`), and video memory preservation is unconfigured.

### Layer 2: NVIDIA Open Kernel Modules
* **Repository**: `github.com/NVIDIA/open-gpu-kernel-modules`
* **Problem 1 (D3cold on DisplayPort Alt Mode)**:
  By default, `NVreg_DynamicPowerManagement` defaults to `0x03` (fine-grained D3cold with video memory power-down). When an external monitor is attached directly to the NVIDIA dGPU over USB-C (DisplayPort Alt Mode) and the monitor enters DPMS off/standby, the driver powers down the GPU silicon into D3cold. When Mutter subsequently sends KMS atomic commits to re-enable or change the CRTC state, the AUX channel communication and atomic modeset block indefinitely waiting on the powered-down GPU.
* **Problem 2 (S0ix Assumption vs Hardware Support)**:
  Even on platforms where the ACPI tables only advertise `[s2idle]` (S0ix), `NVreg_EnableS0ixPowerManagement` defaults to `0` instead of auto-enabling.

### Layer 3: GNOME Mutter (Wayland Compositor)
* **Repository**: `gitlab.gnome.org/GNOME/mutter`
* **Problem (Synchronous Blocking on KMS Thread)**:
  Mutter's `meta-kms-impl-device-atomic.c` executes display configuration and DPMS state transitions. When `gsd-power` invokes `org.gnome.Mutter.DisplayConfig.PowerSaveMode`, Mutter's main thread waits synchronously for the KMS thread to complete the commit.
  If the kernel driver (such as `nvidia-drm`) blocks inside `drmModeAtomicCommit` or returns an unexpected error (`EPERM`), the KMS thread never returns a completion event. Mutter's main event loop deadlocks, and because Mutter also hosts `libinput`, all keyboard and mouse inputs freeze immediately.

### Layer 4: USB & Peripheral Power Management (Wake Trigger)
* **Subsystem**: Linux USB Core / Desktop Environment Wake Policy
* **Role (Trigger, Not Root Cause)**:
  Wireless dongles (such as the Logitech Bolt USB receiver `046d:c548`, generic 2.4GHz receivers, or Bluetooth adapters) default to `power/wakeup: enabled`. Optical mouse sensor noise or RF polling packets trigger wakeup events within seconds of entering `s2idle`, repeatedly bouncing the laptop awake into an uncoordinated GPU state.
* **Important Design Consideration**:
  Disabling USB wakeup via udev (`ATTR{power/wakeup}="disabled"`) disables **both mouse and keyboard wakeups** on shared receivers. Tapping a wireless keyboard when the laptop is docked will no longer wake the system. The graphics freeze itself is an NVIDIA/Mutter failure upon resume; once the display stack is fixed, waking from suspend is fast and reliable. Distinguishing mouse jitter from intentional keyboard wakeups is a feature that belongs in desktop settings (GNOME / COSMIC), not a low-level graphics workaround.

---

## 5. Reliable Reproduction Procedure

To verify the bug or demonstrate the failure on a hybrid system:

### Prerequisites
- Laptop with Intel iGPU + NVIDIA dGPU.
- External monitor connected via USB-C (DisplayPort Alt Mode) directly to the NVIDIA GPU (`/sys/class/drm/card0-DP-*`).
- Fedora / GNOME Wayland session.

### Reproduction A: Testing the DPMS Deadlock (No Suspend Needed)
1. Ensure the default RPM Fusion configuration is active (or ensure `DynamicPowerManagement: 3` is set in `/proc/driver/nvidia/params`).
2. Run a terminal with live journal monitoring on a secondary device or over SSH:
   ```bash
   journalctl -f | grep -E "PowerSaveMode|gnome-shell|nvidia"
   ```
3. Issue a direct DPMS turn-off command via D-Bus:
   ```bash
   busctl --user set-property org.gnome.Mutter.DisplayConfig /org/gnome/Mutter/DisplayConfig org.gnome.Mutter.DisplayConfig PowerSaveMode i 1
   ```
4. **Observed Failure**:
   * The laptop screen turns off.
   * The external USB-C screen freezes displaying the current desktop/lockscreen image.
   * After 25–40 seconds, the journal reports:
     `gsd-power: Error setting property 'PowerSaveMode' on interface org.gnome.Mutter.DisplayConfig: Timeout was reached`
   * Mouse and keyboard cease to function.

### Reproduction B: Testing the Suspend/Resume DRM Master Race
1. Ensure `/usr/lib/systemd/system/systemd-suspend.service.d/nvidia-suspend-nofreeze.conf` is active (unmasked).
2. Lock the screen and trigger suspend:
   ```bash
   loginctl lock-session && systemctl suspend
   ```
3. Immediately wake the machine after 2 seconds by tapping the keyboard or power button.
4. **Observed Failure**:
   * Journal logs `gnome-shell: [atomic] Failed to disable device: drmModeAtomicCommit: Permission denied`.
   * Mutter's KMS worker locks up; screens fail to turn on or external screen freezes on blurred background.

---

## 6. Testing & Verification Matrix

After applying configuration changes or patches, use this matrix to verify that each failure mode has been resolved:

| Test ID | Test Name | Procedure | Expected Pass Criteria | Failure Criteria |
| :--- | :--- | :--- | :--- | :--- |
| **TC-01** | **Module Parameter Validation** | Run `cat /proc/driver/nvidia/params` | `PreserveVideoMemoryAllocations: 1`<br>`EnableS0ixPowerManagement: 1`<br>`DynamicPowerManagement: 1` | Any value set to `0` or Dynamic PM at `3` |
| **TC-02** | **Systemd Freeze Mask Validation** | Run `systemctl cat systemd-suspend.service` | Shows mask for `nvidia-suspend-nofreeze.conf` | `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false` present |
| **TC-03** | **Interactive DPMS Toggle** | Execute DPMS off, wait 5s, execute DPMS on:<br>`busctl --user set-property org.gnome.Mutter.DisplayConfig /org/gnome/Mutter/DisplayConfig org.gnome.Mutter.DisplayConfig PowerSaveMode i 1 && sleep 5 && busctl --user set-property org.gnome.Mutter.DisplayConfig /org/gnome/Mutter/DisplayConfig org.gnome.Mutter.DisplayConfig PowerSaveMode i 0` | Displays turn off cleanly and immediately re-awaken without delay or D-Bus timeout | D-Bus timeout after 25s; external display frozen on old buffer |
| **TC-04** | **System Suspend & Resume Cycle** | Connect external USB-C monitor, run `systemctl suspend`, wait 15s, press power button to wake | System enters `s2idle`, wakes up cleanly, both displays turn on immediately, login prompt accepts input | Black screens, no response to keyboard, hard power cycle required |
| **TC-05** | **USB Sleep Wakeup Immunity** | Put system to sleep, move mouse or tap mouse desk surface | Laptop remains in `s2idle` sleep; does not wake on micro-vibrations | System wakes up within 1–5 seconds without deliberate user intent |
| **TC-06** | **AC Idle Stability** | Leave laptop connected to USB-C display on AC power for 30+ minutes | Screens blank after 5 minutes; moving the mouse instantly wakes both displays | System freezes after 15 minutes of idle |

---

## 7. Immediate Local Workarounds & Automation Script

While upstream fixes are processed, the following configuration changes immediately mitigate both the DPMS screen blanking deadlock and the suspend/resume race condition on the local system.

### Automated Re-Apply Script (One-Liner)

Run the following script with `sudo` (or as `root`) to apply all mitigations automatically on Fedora:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "==> 1. Configuring NVIDIA S0ix, D3hot & Video Memory Preservation..."
cat << 'EOF' > /etc/modprobe.d/nvidia-power-management.conf
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_TemporaryFilePath=/var/tmp
options nvidia NVreg_EnableS0ixPowerManagement=1
options nvidia NVreg_DynamicPowerManagement=0x01
EOF

echo "==> 2. Enabling NVIDIA Sleep Services..."
systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service

echo "==> 3. Masking obsolete SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false drop-ins..."
for service in systemd-suspend systemd-hibernate systemd-hybrid-sleep systemd-suspend-then-hibernate; do
    mkdir -p "/etc/systemd/system/${service}.service.d"
    ln -sf /dev/null "/etc/systemd/system/${service}.service.d/nvidia-suspend-nofreeze.conf"
done
systemctl daemon-reload

echo "==> 4. Disabling Logitech Bolt USB receiver wake-from-sleep jitter..."
cat << 'EOF' > /etc/udev/rules.d/99-disable-bolt-wakeup.rules
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="046d", ATTR{idProduct}=="c548", ATTR{power/wakeup}="disabled"
EOF
udevadm control --reload-rules && udevadm trigger -s usb

echo "==> 5. Disabling automatic suspend on AC power in GNOME..."
if [ -n "${SUDO_USER:-}" ]; then
    sudo -u "$SUDO_USER" dbus-launch gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing' || true
else
    gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing' || true
fi

echo "==> 6. Rebuilding initramfs..."
dracut -f

echo "==> Done! Please reboot the system."
```

### Step-by-Step Manual Instructions

#### Step 1: Prevent Automatic Sleep on AC Power
Prevent GNOME Shell from suspending after 15 minutes while connected to AC power and external desk displays:
```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

#### Step 2: Configure NVIDIA Kernel Module Parameters
Create `/etc/modprobe.d/nvidia-power-management.conf` with:
```bash
sudo tee /etc/modprobe.d/nvidia-power-management.conf << 'EOF'
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_TemporaryFilePath=/var/tmp
options nvidia NVreg_EnableS0ixPowerManagement=1
options nvidia NVreg_DynamicPowerManagement=0x01
EOF
```
* Key parameters:
  - `NVreg_PreserveVideoMemoryAllocations=1`: Retains VRAM across power transitions.
  - `NVreg_TemporaryFilePath=/var/tmp`: Directs memory snapshots to persistent storage instead of tmpfs.
  - `NVreg_EnableS0ixPowerManagement=1`: Synchronizes with ACPI `s2idle`.
  - `NVreg_DynamicPowerManagement=0x01`: Forces D3hot rather than full-chip D3cold (`0x03`), preventing AUX channel modeset deadlocks when waking external USB-C DP Alt Mode screens.

#### Step 3: Enable Systemd NVIDIA Sleep Services
Ensure NVIDIA's helper services handle GPU state during suspend/resume:
```bash
sudo systemctl enable nvidia-suspend.service nvidia-resume.service nvidia-hibernate.service
```

#### Step 4: Mask the Session-Unfreeze Workaround
Mask the legacy `nvidia-suspend-nofreeze.conf` drop-ins so systemd-logind cleanly freezes user sessions before entering sleep, avoiding `drmModeAtomicCommit: Permission denied`:
```bash
sudo mkdir -p /etc/systemd/system/systemd-suspend.service.d \
              /etc/systemd/system/systemd-hibernate.service.d \
              /etc/systemd/system/systemd-hybrid-sleep.service.d \
              /etc/systemd/system/systemd-suspend-then-hibernate.service.d

for s in systemd-suspend systemd-hibernate systemd-hybrid-sleep systemd-suspend-then-hibernate; do
    sudo ln -sf /dev/null "/etc/systemd/system/${s}.service.d/nvidia-suspend-nofreeze.conf"
done

sudo systemctl daemon-reload
```

#### Step 5: Prevent Logitech Bolt Receiver Wakeup Jitter
Disable wake-from-sleep on the Logitech Bolt USB receiver (`046d:c548`) to eliminate immediate sleep bouncing:
```bash
sudo tee /etc/udev/rules.d/99-disable-bolt-wakeup.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="046d", ATTR{idProduct}=="c548", ATTR{power/wakeup}="disabled"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger -s usb
```

#### Step 6: Rebuild Initramfs and Reboot
```bash
sudo dracut -f
sudo reboot
```

### Diagnostic Commands for Future Freezes

If an unexpected freeze or stall occurs, inspect the previous boot log using:
```bash
# Check last 100 log lines of previous boot
journalctl -b -1 -e -n 100

# Check for GNOME Shell unresponsiveness & DPMS timeout
journalctl -b -1 | grep -iE "unresponsive|PowerSaveMode"

# Check kernel messages during suspend/resume transitions
journalctl -b -1 -k | grep -iE "PM: suspend|PM: resume|atomic|nvidia"
```

---

## 8. Upstream Contribution Roadmap & Patch Templates

Here are the concrete proposals and patch templates you can submit to each upstream project:

---

### Contribution 1: RPM Fusion Packaging Fix (*Highest Impact*)
* **Repository**: [github.com/RPM-Fusion/xorg-x11-drv-nvidia](https://github.com/RPM-Fusion/xorg-x11-drv-nvidia)
* **Bug Tracker**: [bugz.rpmfusion.org/xorg-x11-drv-nvidia](https://bugz.rpmfusion.org/xorg-x11-drv-nvidia)
* **Components Affected**: `xorg-x11-drv-nvidia.spec`, `nvidia-power-management.conf`

#### Proposed Patch:
```diff
diff --git a/nvidia-power-management.conf b/nvidia-power-management.conf
index 1234567..abcdefg 100644
--- a/nvidia-power-management.conf
+++ b/nvidia-power-management.conf
@@ -1,7 +1,8 @@
 # Save and restore all video memory allocations.
-#options nvidia NVreg_PreserveVideoMemoryAllocations=1
+options nvidia NVreg_PreserveVideoMemoryAllocations=1
+options nvidia NVreg_TemporaryFilePath=/var/tmp
+options nvidia NVreg_EnableS0ixPowerManagement=1
+options nvidia NVreg_DynamicPowerManagement=0x01
 
 # The destination should not be using tmpfs, so we prefer
 # /var/tmp instead of /tmp
-#options nvidia NVreg_TemporaryFilePath=/var/tmp
diff --git a/xorg-x11-drv-nvidia.spec b/xorg-x11-drv-nvidia.spec
index 2345678..bcdefgh 100644
--- a/xorg-x11-drv-nvidia.spec
+++ b/xorg-x11-drv-nvidia.spec
@@ -450,10 +450,8 @@ rm -rf %{buildroot}
-%{_unitdir}/systemd-suspend.service.d/nvidia-suspend-nofreeze.conf
-%{_unitdir}/systemd-hibernate.service.d/nvidia-suspend-nofreeze.conf
-%{_unitdir}/systemd-hybrid-sleep.service.d/nvidia-suspend-nofreeze.conf
-%{_unitdir}/systemd-suspend-then-hibernate.service.d/nvidia-suspend-nofreeze.conf
+# Dropped obsolete nvidia-suspend-nofreeze.conf:
+# Setting SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false causes a fatal race condition
+# under Wayland where GNOME Shell loses DRM Master during suspend, throwing
+# 'drmModeAtomicCommit: Permission denied' and deadlocking Mutter.
```

#### Bug Report / PR Rationale:
> **Subject**: Drop `nvidia-suspend-nofreeze.conf` and update power management defaults for Wayland / modern laptops
> 
> **Description**:
> The drop-in `nvidia-suspend-nofreeze.conf` sets `SYSTEMD_SLEEP_FREEZE_USER_SESSIONS=false`. While historically useful for Xorg VT switching, this causes severe deadlocks on Wayland: GNOME Shell continues issuing atomic KMS commits concurrently as `systemd-logind` revokes DRM Master on suspend. This produces `drmModeAtomicCommit: Permission denied` and locks up Mutter's KMS worker thread.
> 
> Furthermore, on modern laptops that only support `s2idle` (S0ix), shipping `/usr/lib/modprobe.d/nvidia-power-management.conf` with commented-out parameters leads to corrupted GPU state on resume and DPMS hangs on external USB-C displays.
> 
> Enabling `NVreg_PreserveVideoMemoryAllocations=1`, `NVreg_EnableS0ixPowerManagement=1`, and `NVreg_DynamicPowerManagement=0x01` out of the box prevents display freezes across hybrid Intel/NVIDIA systems.

---

### Contribution 2: NVIDIA Open Kernel Driver Fix
* **Repository**: [github.com/NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules)
* **Area**: `kernel-open/nvidia-drm`, `kernel-open/nvidia`

#### Issue Submission:
> **Title**: Deadlock in `drmModeAtomicCommit` when waking USB-C DP Alt Mode displays from D3cold (`NVreg_DynamicPowerManagement=3`)
>
> **Hardware**:
> - Integrated: Intel Arrow Lake-S Graphics (`i915`)
> - Discrete: NVIDIA RTX PRO 1000 Blackwell / GB207GLM (`nvidia-drm`)
> - External display connected via USB-C DisplayPort Alt Mode directly to NVIDIA dGPU.
>
> **Problem Description**:
> Under GNOME Wayland with dual GPUs, setting `PowerSaveMode` (DPMS Off) powers off the external display. With `NVreg_DynamicPowerManagement=3`, the NVIDIA GPU enters D3cold.
> 
> When Mutter attempts an atomic commit to change display power states or re-enable the display, the ioctl in `nvidia-drm` blocks indefinitely waiting for the GPU to power up and complete link training, deadlocking Mutter's KMS thread and timing out D-Bus callers.
> 
> **Workaround**:
> Setting `NVreg_DynamicPowerManagement=0x01` (D3hot) completely eliminates the hang.
> 
> **Proposed Resolution**:
> 1. Inhibit transitioning to D3cold when display connectors are in use or registered as connected in DRM KMS, falling back to D3hot.
> 2. Auto-enable `NVreg_EnableS0ixPowerManagement=1` when the platform firmware only supports `s2idle`.

---

### Contribution 3: GNOME Mutter Resilience Fix
* **Repository**: [gitlab.gnome.org/GNOME/mutter](https://gitlab.gnome.org/GNOME/mutter/-/issues)
* **Area**: `src/backends/native/meta-kms-impl-device-atomic.c`

#### Issue Submission:
> **Title**: Mutter atomic KMS state machine fails to recover when `drmModeAtomicCommit` returns `EPERM` during session pause
>
> **Problem Description**:
> In multi-GPU setups (e.g. Intel primary driving internal eDP, NVIDIA secondary driving external DisplayPort over USB-C), when `systemd-logind` pauses a session or initiates system suspend, it revokes DRM Master.
> 
> If an atomic modesetting commit or power state transition is in-flight or dispatched concurrently, `drmModeAtomicCommit` fails with `drmModeAtomicCommit: Permission denied` (`-EPERM`).
> 
> Currently, Mutter's atomic backend in `meta-kms-impl-device-atomic.c` does not recognize `EPERM` as a transient session-pause event. The KMS worker thread retains pending/in-flight commit state, leaving the display device in an inconsistent state that fails to recover or process page flips when the session is resumed, contributing to a permanent freeze of the compositor.
> 
> **Proposed Improvement**:
> 1. Treat `EPERM` / `EACCES` from `drmModeAtomicCommit` as an expected error condition when the session is pausing or losing DRM Master.
> 2. Reset in-flight atomic state cleanly and defer further commits until DRM Master is regained upon session reactivation.

---

### Contribution 4: Desktop Environment Peripheral Wake Policy (Future Feature)
* **Target Projects**: [GNOME Control Center](https://gitlab.gnome.org/GNOME/gnome-control-center) / [COSMIC Settings](https://github.com/pop-os/cosmic-settings)
* **Area**: Power Management & Input Device Settings

#### Feature Proposal:
> **Subject**: User-configurable peripheral wakeup policy (Keyboard vs. Mouse wake)
> 
> **Rationale**:
> Many users connect their laptops to external docks or desk monitors and suspend the machine with the lid closed. Waking the computer by pressing a key on an external wireless keyboard is a standard and expected workflow.
> 
> However, modern optical mice and composite wireless receivers (like the Logitech Bolt `046d:c548`) can emit micro-vibration or RF polling events that trigger unwanted resume cycles within seconds of entering sleep.
> 
> Disabling USB wakeup globally at the udev level breaks keyboard wakeups for all users. A desktop setting in GNOME / COSMIC allowing users to configure which input devices can wake the system—or filtering mouse sensor jitter while permitting keypress events—would provide an intuitive, user-controlled solution.
