# Jetson Nano eMMC Flash Guide (NixOS Host)

## Context
Flashing a Waveshare Jetson Nano Dev Kit (eMMC variant, module p3448-0002) from a NixOS host using a rootful Podman container.

## NixOS Prerequisites

Add to your NixOS config and rebuild:

```nix
# nix/virtualization/binfmt.nix
{...}: {
  boot.binfmt.emulatedSystems = [ "aarch64-linux" ];
  # preferStaticEmulators registers binfmt with the F (fix-binary) flag,
  # which pre-loads the interpreter so it works inside containers/chroots.
  boot.binfmt.preferStaticEmulators = true;
}
```

After `nixos-rebuild switch`, verify the `F` flag is set:
```sh
cat /proc/sys/fs/binfmt_misc/aarch64-linux
# Should show: flags: PF
```

Copy the static QEMU binary where the L4T script will find it:
```sh
cp /run/binfmt/aarch64-linux /home/matthew/JETSON_NANO/qemu-aarch64-static
chmod +x /home/matthew/JETSON_NANO/qemu-aarch64-static
```

The `nv-apply-debs.sh` script checks `${L4T_DIR}/../qemu-aarch64-static` first, so placing it at the parent of `Linux_for_Tegra/` means it's found automatically.

## File Layout

Download from NVIDIA and extract into `/home/matthew/JETSON_NANO/`:
- `Jetson-210_Linux_R32.7.5_aarch64.tbz2` — BSP
- `Tegra_Linux_Sample-Root-Filesystem_R32.7.5_aarch64.tbz2` — rootfs
- `overlay_32.7.5_PCN211181.tbz2` — overlay patch

```sh
cd /home/matthew/JETSON_NANO
tar -xjf Jetson-210_Linux_R32.7.5_aarch64.tbz2
cd Linux_for_Tegra/rootfs
tar -xjpf ../../Tegra_Linux_Sample-Root-Filesystem_R32.7.5_aarch64.tbz2
cd /home/matthew/JETSON_NANO
tar -xjf overlay_32.7.5_PCN211181.tbz2
```

## Fix SD Card Device Tree (Before Flashing)

The SD card slot (`sdhci@700b0400`) is disabled by default in the Waveshare board's DTB. Fix it using `fdtput` directly on the binary — do NOT use the decompile/recompile approach as NVIDIA's DTBs have too many warnings that cause modern `dtc` to fail.

```sh
sudo nix-shell -p dtc --run /tmp/fix-dtb.sh
```

Write `/tmp/fix-dtb.sh`:
```sh
#!/bin/sh
BASE="/home/matthew/JETSON_NANO/Linux_for_Tegra"
for dtb in \
  "$BASE/kernel/dtb/tegra210-p3448-0002-p3449-0000-b00.dtb" \
  "$BASE/bootloader/tegra210-p3448-0002-p3449-0000-b00.dtb" \
  "$BASE/bootloader/kernel_tegra210-p3448-0002-p3449-0000-b00.dtb" \
  "$BASE/rootfs/boot/kernel_tegra210-p3448-0002-p3449-0000-b00.dtb"; do
    fdtput -ts "$dtb" /sdhci@700b0400 status okay && echo "OK: $dtb"
done
echo "Verify:"
fdtget "$BASE/kernel/dtb/tegra210-p3448-0002-p3449-0000-b00.dtb" /sdhci@700b0400 status
```

Verify output says `okay`.

> **Warning:** Do NOT use `sed -i 's/status = "disabled";/status = "okay";/'` on the decompiled DTS — it changes ALL disabled nodes and will break ethernet and USB on the Waveshare carrier board. Always use `fdtput` for targeted edits.

## Podman Container Setup

```sh
sudo podman run -it --name jetson-flash \
  --privileged \
  --network=host \
  -v /dev/bus/usb:/dev/bus/usb \
  -v /home/matthew/JETSON_NANO:/flash \
  ubuntu:18.04 bash
```

Inside the container:
```sh
apt-get update
apt-get install -y python3 usbutils
```

To re-enter an existing container (preserves installed packages):
```sh
sudo podman start -ai jetson-flash
```

## apply_binaries.sh

```sh
cd /flash/Linux_for_Tegra && ./apply_binaries.sh
```

If it fails with `mknod: File exists`:
```sh
rm -f /flash/Linux_for_Tegra/rootfs/dev/random /flash/Linux_for_Tegra/rootfs/dev/urandom
./apply_binaries.sh
```

## Recovery Mode

For the Waveshare Jetson Nano Dev Kit:
1. Power off
2. Hold **FC REC** button (short FC_REC to GND on header)
3. Apply power via barrel jack (5V/4A)
4. Hold 2 seconds, release
5. Verify: `lsusb | grep -i nvidia` → should show `NVIDIA Corp. APX (0955:7f21)`

> Use a known-good **data-capable** Micro-USB cable. Charge-only cables cause `error -71` in dmesg and the device drops off the bus immediately.

## Flash

```sh
USER=root ./flash.sh jetson-nano-emmc mmcblk0p1
```

Note: `flash.sh` checks `$USER` env var (not actual UID), so `USER=root` is required even when running as root inside the container.

## Post-Flash Setup

Default credentials: `nvidia` / `nvidia`

### Mount SD Card as /workspace

```sh
sudo mkfs.ext4 /dev/mmcblk1p1
sudo mkdir /workspace
sudo mount /dev/mmcblk1p1 /workspace
echo '/dev/mmcblk1p1 /workspace ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

### Install CUDA Toolkit

```sh
sudo apt update && sudo apt install -y cuda-toolkit-10-2
echo 'export PATH=$PATH:/usr/local/cuda/bin' >> ~/.bashrc
source ~/.bashrc
nvcc --version
```

### Demo Commands

```sh
cat /etc/nv_tegra_release        # L4T version
uname -a                          # aarch64 Linux
lsblk                             # eMMC + SD card
df -h /workspace                  # available storage
nvcc --version                    # CUDA 10.2
sudo tegrastats                   # live GPU/CPU/memory
# Compile and run deviceQuery for CUDA core count (128 cores):
cd /usr/local/cuda/samples/1_Utilities/deviceQuery && sudo make && ./deviceQuery
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `chroot: failed to run command 'dpkg'` | Rootfs not extracted | Extract `Tegra_Linux_Sample-Root-Filesystem_*.tbz2` into `rootfs/` |
| `ERROR qemu not found` | No QEMU binary at expected path | Copy `/run/binfmt/aarch64-linux` to `JETSON_NANO/qemu-aarch64-static` |
| `flash.sh requires root privilege` | `$USER` env not set | Run as `USER=root ./flash.sh ...` |
| APX appears then drops off USB | Bad cable | Use a short, known-good data-capable Micro-USB cable |
| SD card not detected (`mmcblk1` missing) | DTB has sdhci disabled | Use `fdtput` to set `/sdhci@700b0400 status okay` in all DTB copies |
| Ethernet/USB broken after DTB edit | Used global sed on DTS | Reflash and use `fdtput` for targeted edits only |
