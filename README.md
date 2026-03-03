# amdisp4 — AMD ISP4 Suspend/Resume Fix for CachyOS

Out-of-tree module that fixes the AMD ISP4 (Image Signal Processor v4.1.1) suspend/resume hang on CachyOS Linux. Tested on the HP ZBook Ultra G1a with AMD Ryzen AI MAX+ PRO 395 (Strix Halo).

Forked from [idovitz/amdisp4](https://github.com/idovitz/amdisp4), which provides the original patches from the [upstream ISP4 patchset](https://lore.kernel.org/lkml/20251128091929.165272-1-Bin.Du@amd.com/).

## What This Fixes

The in-tree `amd_isp4_capture` module shipped with CachyOS 6.19 works — the camera streams video. But if the module has been loaded at any point during your session, closing the laptop lid (s2idle suspend) hangs the system. Hard power-off required.

The in-tree driver has **no PM suspend/resume ops**. When the system enters s2idle, the ISP hardware is left in an undefined state and the SMU can't sequence the power domains on resume. This is also missing from the latest [upstream v9 patchset](https://lkml.org/lkml/2026/3/2/278) on LKML.

This module adds `isp4_capture_suspend()` / `isp4_capture_resume()` — ~50 lines that tear down the ISP firmware and hardware state before sleep via `isp4sd_pwroff_and_deinit()`, preventing the SMU hang.

## What We Changed

The source is identical to the CachyOS in-tree ISP4 driver at `drivers/media/platform/amd/isp4/`, with two additions:

- **PM suspend/resume ops** (`dev_pm_ops`) — `isp4_capture_suspend()` checks if the ISP is powered on, tears down firmware and hardware state via `isp4sd_pwroff_and_deinit()`, and marks the device for re-open on resume
- **Out-of-tree API adaptation** — internal `amdgpu_bo_*` functions replaced with exported `isp_kernel_buffer_alloc/free` and `isp_user_buffer_alloc/free` from `<drm/amd/isp.h>`, enabling building outside the kernel tree

Everything else — IRQ handling (`IRQF_NO_AUTOEN`), the V4L2 subdev, video node, firmware interface — is unchanged from the in-tree version.

The module installs to `/updates/dkms/` which automatically overrides the in-tree version.

## Quick Install (Pre-built Module)

A pre-built `amd_capture.ko.zst` is included for **CachyOS 6.19.5-3-cachyos**:

```bash
sudo cp amd_capture.ko.zst /lib/modules/$(uname -r)/updates/dkms/
sudo depmod
sudo reboot
```

## Building From Source

Requires kernel headers and clang (CachyOS kernels are built with clang):

```bash
sudo pacman -S --needed linux-cachyos-headers base-devel clang llvm

git clone https://github.com/kinncj/amdisp4.git
cd amdisp4
make -C /lib/modules/$(uname -r)/build M=$(pwd) CC=clang LLVM=1 modules
```

Then install:

```bash
zstd amd_capture.ko -o amd_capture.ko.zst
sudo cp amd_capture.ko.zst /lib/modules/$(uname -r)/updates/dkms/
sudo depmod
sudo reboot
```

## Verification

```bash
# Module loaded
lsmod | grep amd

# No ISP errors
dmesg | grep -i isp

# Video device exists
v4l2-ctl --list-devices

# Camera works
mpv av://v4l2:/dev/video0

# Suspend/resume works
systemctl suspend
```

## Troubleshooting

**Build fails with gcc errors**: CachyOS kernels require `CC=clang LLVM=1`. Do not use gcc.

**Module won't load**: Check for a blacklist at `/etc/modprobe.d/blacklist-amd-capture.conf`. Remove or comment out any `blacklist amd_capture` or `install amd_capture /bin/false` lines, then `sudo mkinitcpio -P` and reboot.

**System hangs at boot**: Boot with `nomodeset`, remove the module, reinstall the kernel:
```bash
sudo rm -f /lib/modules/$(uname -r)/updates/dkms/amd_capture.ko.zst
sudo depmod
sudo pacman -S linux-cachyos linux-cachyos-headers
```

**Pre-built module doesn't load**: The `.ko.zst` is built for a specific kernel version. If yours differs, build from source.

## Kernel Updates

The module must be rebuilt for each kernel update. Either rebuild manually or set up DKMS using the included `dkms.conf` and `Makefile`.

## License

GPL-2.0 — same as the upstream Linux kernel ISP4 driver.
