# nvidia-steam-install

Turn a **clean SteamOS OOBE repair image** into a **one-click USB installer**
with **NVIDIA (RTX) 575 driver support baked in** — so you can install SteamOS
onto any PC with a modern NVIDIA card, not just a Steam Deck.

Built and known-good against **SteamOS 3.8**, which ships the
**`nvidia-open` 575.64.05** driver (open kernel modules). The script does not
pin a version — it builds whatever `nvidia-open-dkms` the image's own SteamOS
branch serves, so a 3.8 image yields the 575.64.05 release.

```bash
sudo ./steamos-nvidia-installer.sh steamdeck-oobe-repair-<ver>.img
```

Output: `<image>-nvidia-usbinstall.img` → `dd` it to a USB stick, boot the
target machine (UEFI, Secure Boot **off**), double-click **"Install SteamOS
(NVIDIA) to Hard Drive"**, pick a disk, done. The input image is copied first
and never modified.

## What it does

In one pass over a single copy of the image:

1. **Builds the NVIDIA driver** — compiles `nvidia-open` (DKMS) against the
   image's exact `neptune` kernel inside a throwaway overlayfs chroot, using
   Valve's frozen Arch mirror. The toolchain and headers never enter the
   image; only the driver payload (kernel modules, `nvidia-utils`, `lib32`,
   `egl-*`, GSP firmware) is copied into the rootfs and registered in the
   pacman database.
2. **Enables it at boot** — blacklists `nouveau` and turns on `nvidia-drm`
   KMS via both `modprobe.d` and the kernel cmdline (`grub.cfg` on the EFI
   partition and `/etc/default/grub`, so the installed system's regenerated
   grub keeps it).
3. **Makes OS updates self-healing** *(default)* — updating from within Steam
   works normally: Valve's updater stages the new OS in the spare A/B slot,
   then a wrapper around `steamos-update` rebuilds the NVIDIA driver for the
   new OS (in a chroot on the new slot, from **that** version's own repo
   branch) before the reboot prompt appears. If the rebuild fails, the update
   is cancelled and the machine keeps booting the current working system.
4. **Adds the one-click installer** — Valve's own `repair_device.sh` (which
   installs by *cloning* the running system, so the driver propagates),
   patched for generic hardware: target-disk override, `/dev/sdX`
   partition-suffix autodetect, NVMe-sanitize skipped on non-NVMe media —
   plus a zenity disk-picker, a desktop icon, and NOPASSWD sudo for `deck`.

## Options

| Flag | Effect |
|------|--------|
| *(default)* | Self-healing updates — rebuilds the driver into the new A/B slot on each OS update. |
| `--hold-updates` | Hard-hold OS updates instead (Steam always shows "up to date"). |
| `--no-hold-updates` | Stock update behaviour — **DANGER:** an OS update boots an unpatched system (A/B fallback saves you, but the driver is lost). |
| `--no-installer` | Skip step 4 (produce a plain bootable patched OS). |
| `--trim-cuda` | Drop CUDA / OpenCL / NVVM / OptiX libs (~350 MB smaller). |
| `--skip-sigcheck` | Disable pacman signature checks in the build chroot. |
| `--workdir DIR` | Build dir (~3 GB; default: alongside the output). Kept between runs to cache the driver build. |

## Requirements

- **Host:** Arch-ish Linux with `losetup`, `btrfs-progs`, `rsync`, `curl`,
  `kmod`, `zstd`.
- **Driver support:** `nvidia-open` covers **RTX 20xx (Turing) and newer only**
  — no Maxwell / Pascal / Volta.
- **Target machine:** UEFI with **Secure Boot off**.

First boot of an installed system lands in the gamescope Steam setup. If it
black-screens: `Ctrl+Alt+F3` → `steamos-session-select plasma`.

## Related

Sibling project: [`steamos-nvidia-installer`](https://github.com/28allday/steamos-nvidia-installer)
— the tracking-latest build. This repo is the **575-driver known-good** variant.
