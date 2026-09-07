# fydeos-sheng

> **Project Status**
> Early development stage. Functionality may be incomplete and build results may contain unknown issues.

---

## Overview

**fydeos-sheng** ports [FydeOS](https://fydeos.com)(Chromium OS based) to the **Xiaomi Pad 6S Pro (sheng)**.

It takes the official [FydeOS for Raspberry Pi 5 / 500 / 500+](https://fydeos.com/download/device/rpi5-fydeos/) image, keeps the FydeOS rootfs (ChromeOS userspace with the browser and web apps), replaces the BCM2712 kernel with the sheng mainline kernel ([ianchb/sm8550-mainline](https://github.com/ianchb/sm8550-mainline)), and adds the device firmware — producing a flashable `rootfs.img`, a `state.img` (stateful user data) and a `boot.img` for the Qualcomm bootloader.

The boot flow follows the original Raspberry Pi design of FydeOS: boot the kernel directly with

```
root=PARTLABEL=<partition> rootwait ro cros_debug cros_secure chromiumos.allow_overlayfs console=tty0
```

(no initramfs; the ChromeOS rootfs runs in overlayfs read-only mode, like on the RPi).

---

## Getting Started

### 1. Run the Build Workflow

1. Open the **Actions** tab of your fork/repository.
2. Select **Build FydeOS RootFS**, click **Run workflow** (defaults: dual-boot to partition `linux`, prebuilt kernel).
3. Download the artifacts:
   - **`rootfs-fydeos-linux`** — the FydeOS rootfs (ext4, ~2.7 GB)
   - **`state-fydeos`** — stateful partition (ext4, label `STATE`, default 8 GiB)
   - **`boot-fydeos-linux`** — `boot.img` (sheng kernel + FydeOS cmdline, `console=tty0`)

### 2. Prepare Partitions (TWRP)

Prerequisites: unlocked bootloader, TWRP, fastboot. Inside TWRP, using `parted`, shrink `userdata` and create **two** partitions at the end:

| Partition name | Size | Content |
|---|---|---|
| `linux` | ≥ 3 GiB (or `userdata` in single-boot mode) | FydeOS rootfs |
| `state` | 8+ GiB | STATE (user data, ext4 label `STATE`) |

### 3. Flash

```bash
# Dual boot (slot B)
fastboot erase dtbo_b
fastboot flash boot_b boot.img
fastboot flash linux rootfs.img
fastboot flash state state.img
fastboot reboot
```

### 4. First Boot

- The kernel cmdline of the repacked `boot.img` shows boot logs on the screen (`console=tty0`, `loglevel=7`).
- FydeOS boots to the **developer-mode overlayfs flow** — first boot may take a while, the ChromeOS logo appears on the panel, followed by the OOBE setup.
- After setup, the Web UI is FydeOS desktop; **SSH on port 22** is available in developer mode (`ssh root@<ip>` — the root password is empty/dev mode password is unset by default on the fist boot).

---

## How It Works (Build Pipeline)

1. **Download** the official FydeOS SBC image (`FydeOS_for_SBC_Pi5_v22.1-com.bin.zip`, ~2.2 GB) and extract the disk image (~9.5 GB).
2. **Detect** the `ROOT-A` partition with `losetup -P` + `blkid` (label `ROOT-A`, ext4).
3. **Modify in place**:
   - remove the BCM2712 kernel modules (`/lib/modules/*`);
   - extract `linux-xiaomi-sheng.deb` into the rootfs (boot files + sheng modules) and run `depmod`;
   - copy `ianchb/sheng-firmware` into `/usr/lib/firmware` (ath12k/WCN7850 Wi-Fi 7, qca Bluetooth, cirrus DSP, qcom sm8550).
4. **Extract** the modified partition out of the disk image as `rootfs.img` (sector-exact `dd`, offsets from `/sys/class/block/loopXpN`).
5. **Create** `state.img` (ext4, label `STATE`, sparse).
6. **Repack `boot.img`**: extract the kernel payload from the prebuilt `boot_sheng_*` image (or use a custom-built kernel) and run `mkbootimg` with the FydeOS cmdline (`console=tty0` for visible logs; `quiet` only if the input is enabled).
7. **Upload** the three artifacts.

---

## Advanced Configuration

| Parameter | Description | Default |
|---|---|---|
| **FydeOS image URL** | Official SBC image download URL | `https://download.fydeos.io/v22.1/FydeOS_for_SBC_Pi5_v22.1-com.bin.zip` |
| **Boot mode** | `single (userdata)` / `dual (linux)` / `custom` | `dual (linux)` |
| **Quiet boot** | Adds `quiet` to the kernel cmdline | `true` |
| **Kernel source** | `prebuilt` (ianchb/sm8550-mainline) / `custom_build` | `prebuilt` |
| **Kernel Repo / Branch / Config** | Custom kernel build inputs | `ianchb/sm8550-mainline` / `sheng-7.2.2` / `sm8550.config` |
| **Firmware Repo / Branch** | Device firmware source | `ianchb/sheng-firmware` / `master` |
| **State size** | Stateful image size in MiB | `8192` |

---

## Known Limitations

- The FydeOS (Chromium OS) userspace was built for Raspberry Pi 5; on sheng (SM8550) the DRM panel and GPU are exposed to Chrome via the mainline `msm` driver. Graphics acceleration may fall back to software rendering; if the UI stays black or crashes, add `--use-gl=swiftshader` in `/etc/chrome_dev.conf` inside the rootfs.
- Wi-Fi uses `ath12k` (WCN7850) and the sheng firmware — should be picked up by shill; otherwise configure networking via `eth0` with a USB-C ethernet adapter.
- Only the dual/single/custom partition-boot flows of this repo are supported; the ChromeOS verified-boot chain is not used (developer-mode overlayfs flow).
- The `STATE` image is created empty (format will be initialized by ChromeOS on first boot).

---

## Credits

- [FydeOS](https://fydeos.com) — the operating system and the SBC image.
- [ianchb/debian-sheng](https://github.com/ianchb/debian-sheng) — sheng kernel/firmware sources, mkbootimg, build patterns; [map220v](https://github.com/map220v) — mainline kernel port & TWRP.
- [ianchb/sm8550-mainline](https://github.com/ianchb/sm8550-mainline) — prebuilt sheng kernel releases.
- [ianchb/sheng-firmware](https://github.com/ianchb/sheng-firmware) — device firmware blobs.
