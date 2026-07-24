# nvidia-steam-install

Turn a **clean SteamOS OOBE repair image** into a **one-click USB installer**
with **NVIDIA (RTX) 575 driver support baked in** — so you can install SteamOS
onto any PC with a modern NVIDIA card, not just a Steam Deck.

This is the **older-driver** edition — it works exactly like
[`steamos-nvidia-installer`](https://github.com/28allday/steamos-nvidia-installer)
but targets the **`nvidia-open` 575 series** (575.64.05, open kernel modules),
as shipped by **SteamOS 3.8**. The script does not pin a version — it builds
whatever `nvidia-open-dkms` the image's own SteamOS branch serves, so a 3.8
image yields the 575.64.05 release. Need the newer drivers? Use
`steamos-nvidia-installer` instead.

## Getting started

You run the script **on an Arch-ish Linux host** (see [Requirements](#requirements)).
It produces a USB image; you then flash that image and boot it on the target PC.

**1. Get the script**

```bash
git clone https://github.com/28allday/nvidia-steam-install.git
cd nvidia-steam-install
chmod +x steamos-nvidia-installer.sh
```

**2. Download a clean SteamOS recovery image**

Grab the official **SteamOS recovery (OOBE repair) image** from Valve:
<https://store.steampowered.com/steamos/download/?ver=steamdeck&snr=>

Unzip it so you have a `.img` file (e.g. `steamdeck-repair-<ver>.img`). Use a
**clean, unmodified** image — not one that has already been patched.

**3. Build the NVIDIA installer image**

```bash
sudo ./steamos-nvidia-installer.sh steamdeck-repair-<ver>.img
```

The input image is **copied first and never modified**. The build compiles the
driver and takes a few minutes; it needs ~3 GB of free space for the work dir.
Output lands next to the input as:

```
steamdeck-repair-<ver>-nvidia-usbinstall.img
```

**4. Flash it to a USB stick**

Find your USB device with `lsblk`, then write the image (this **erases** the
stick — double-check the device node):

```bash
sudo dd if=steamdeck-repair-<ver>-nvidia-usbinstall.img of=/dev/sdX bs=4M status=progress oflag=sync
```

(Or use a GUI tool such as GNOME Disks / balenaEtcher / Ventoy.)

**5. Install on the target PC**

Boot the USB stick on the target machine — **UEFI, with Secure Boot off**.
On the desktop, double-click **"Install SteamOS (NVIDIA) to Hard Drive"**,
pick a disk, and let it run. Done.

> First boot of the installed system lands in the gamescope Steam setup. If it
> black-screens: `Ctrl+Alt+F3` → `steamos-session-select plasma`.

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

## Related

This is a sibling of [**`steamos-nvidia-installer`**](https://github.com/28allday/steamos-nvidia-installer)
— the main, current-driver build. **This repo works exactly the same way; it
just supports older NVIDIA drivers** (the `nvidia-open` 575 series, as shipped
by SteamOS 3.8). Use this one if you need the older driver; use
`steamos-nvidia-installer` for the newer drivers.
