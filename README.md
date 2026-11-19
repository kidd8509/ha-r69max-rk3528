# Home Assistant OS port for R69MAX / RK3528

Home Assistant Operating System (HAOS) board port for the R69MAX TV box based on Rockchip RK3528. The intended hardware variant is **2 GB RAM / 16 GB eMMC**; verify the actual PCB, DDR, storage, Wi-Fi and Ethernet revisions before flashing.

> **Project status (2026-11-19):** This repository currently contains only its initial README on `main`. The board-port source on the local `rk3528-r69max-2-16` branch has **not yet been published here**, so the exact board defconfig, bootloader artifacts, image name, and SD/eMMC boot procedures have not been verified. This document is a reproducible setup and test guide, **not a claim that the device already boots HAOS**. Do not flash until those items are confirmed.

## 1. Versions and source layout

| Item | Value |
|---|---|
| Upstream | [home-assistant/operating-system](https://github.com/home-assistant/operating-system) |
| Pinned upstream HAOS commit | [`c7acf719b948b192ee11dd296f0cdb331e33f0c0`](https://github.com/home-assistant/operating-system/commit/c7acf719b948b192ee11dd296f0cdb331e33f0c0) |
| R69MAX repository | [kidd8509/ha-r69max-rk3528](https://github.com/kidd8509/ha-r69max-rk3528) |
| Target board branch | `rk3528-r69max-2-16` (initially local; push it separately) |
| Build system | HAOS Buildroot with the pinned `buildroot` submodule |
| Expected outputs | Full-disk `.img.xz` image and, if configured, RAUC `.raucb` update bundle |

The port should include, at minimum, a board-specific `buildroot-external/configs/<target>_defconfig`, board files, the appropriate RK3528 kernel/device tree and bootloader/image-generation definitions. **The upstream commit does not itself provide an R69MAX target**.

## 2. Prepare the build environment

**Recommended host:** x86-64 Linux (Ubuntu/Debian) with rootful Docker. The upstream HAOS build invokes a **privileged** Docker container and needs usable loop devices. Rootless Docker is not supported for creating the complete disk image. Use a regular non-root user with Docker access; the HAOS entry script refuses to run as root.

Install Git, Docker Engine, rsync and xz utilities using your distribution's package manager. For Ubuntu, after installing/configuring Docker Engine:

```bash
sudo apt update
sudo apt install -y git rsync xz-utils ca-certificates curl
docker version
docker info
ls -l /dev/loop-control
```

Keep ample free disk space for Buildroot sources, compiler output and the Docker/cache directories. The first build is substantially larger/slower than subsequent builds. If working on macOS, use a Linux build host or a Linux VM in which privileged Docker and loop devices are verified; a native macOS shell is **not** an equivalent HAOS build environment.

## 3. Download the pinned Home Assistant OS base

```bash
mkdir -p "$HOME/src"
cd "$HOME/src"
git clone https://github.com/home-assistant/operating-system.git haos-r69max-base
cd haos-r69max-base
git checkout --detach c7acf719b948b192ee11dd296f0cdb331e33f0c0
git submodule update --init --recursive
git rev-parse HEAD
git submodule status
```

Expected HEAD: `c7acf719b948b192ee11dd296f0cdb331e33f0c0`. Do not replace the pinned submodule with an arbitrary Buildroot revision.

## 4. Overlay the R69MAX port onto the HAOS base

The following is for a **separate port/overlay repository**. `PORT_DIR` must point to your **actual local** `rk3528-r69max-2-16` working tree; it is not yet cloneable from the remote until that branch is pushed.

```bash
BASE_DIR="$HOME/src/haos-r69max-base"
PORT_DIR="/absolute/path/to/your/local/rk3528-r69max-2-16"

git -C "$PORT_DIR" branch --show-current
git -C "$PORT_DIR" status --short
test -d "$PORT_DIR/buildroot-external" || {
  echo "Missing port source: buildroot-external"; exit 1;
}
rsync -av "$PORT_DIR/buildroot-external/" "$BASE_DIR/buildroot-external/"
# If the port also changes HAOS scripts, copy only the reviewed changes:
# rsync -av "$PORT_DIR/scripts/" "$BASE_DIR/scripts/"

cd "$BASE_DIR"
git status --short
find buildroot-external/configs -maxdepth 1 -name '*_defconfig' -print
```

Do **not** use `rsync --delete` or replace the upstream `.git` directory. Examine any port changes to `Makefile`, `Dockerfile`, `.gitmodules`, bootloader generation, and signing before bringing them across.

If your local branch is instead a **complete HAOS fork** already based on the pinned commit, build in that checkout directly after `git submodule update --init --recursive`; do not overlay the whole fork onto another Git tree.

## 5. Select the actual R69MAX build target

HAOS targets are named from `buildroot-external/configs/<target>_defconfig`. Find the target added by the port:

```bash
cd "$HOME/src/haos-r69max-base"
ls buildroot-external/configs/*_defconfig
# Replace this example placeholder with the filename's prefix:
TARGET="<r69max-port-target>"
test -f "buildroot-external/configs/${TARGET}_defconfig" || {
  echo "R69MAX board defconfig missing. Stop: port not complete."; exit 1;
}
```

Never assume that upstream `generic_aarch64` produces an RK3528 TV-box boot image: a compatible U-Boot/DDR initialization, DTB and storage layout must be supplied by the port.

## 6. Compile the SD-card image

```bash
cd "$HOME/src/haos-r69max-base"
scripts/enter.sh make O=output_r69max_sd "$TARGET"
find output_r69max_sd/images -maxdepth 1 -type f -print
```

Locate the **full-disk** `.img.xz` artifact (not an individual root filesystem or a `.raucb` bundle). Check that its board name, bootloader layout and device-tree configuration match this R69MAX hardware.

### Write and boot from SD card

**Destructive operation — all data on the selected card will be erased.** On a Linux flashing host, insert the microSD card and positively identify the **whole-disk** block device, not a partition:

```bash
lsblk -o NAME,PATH,SIZE,MODEL,TRAN,TYPE,MOUNTPOINTS
# Example placeholders only; DO NOT run with these unchanged:
IMAGE="/absolute/path/to/haos_r69max-<version>.img.xz"
SD_DEV="/dev/sdX"
```

After manually verifying both variables, that no target partitions are mounted, and that `SD_DEV` is the removable SD card:

```bash
xz -t "$IMAGE"
xzcat "$IMAGE" | sudo dd of="$SD_DEV" bs=4M status=progress conv=fsync
sync
```

Eject safely; insert the card into the R69MAX. Connect Ethernet and a serial console if available. Power on and confirm that the **board-specific bootloader actually boots from SD**, loads the right DTB/kernel, finds the HAOS partitions, and reaches the host shell. If the stock eMMC bootloader takes precedence, inspect its boot order before changing firmware.

## 7. Compile and install to eMMC

Use the **same verified full-disk image** for eMMC only if the port's partitioning and bootloader configuration explicitly support both SD and eMMC. Otherwise build a separate eMMC target:

```bash
# Only when the port supplies a dedicated <emmc-target>_defconfig:
# scripts/enter.sh make O=output_r69max_emmc "<emmc-target>"
```

**Prerequisites:** successful SD-card boot; known working UART recovery method; tested Rockchip MaskROM/loader or rescue-Linux recovery route; full eMMC backup stored off-device; confirmed RK3528 bootloader offsets and eMMC boot order.

Prefer a Linux rescue environment booted from **external media**, with the internal eMMC neither mounted nor used as the running system. Identify it carefully:

```bash
lsblk -o NAME,PATH,SIZE,MODEL,TYPE,MOUNTPOINTS
findmnt -no SOURCE /
# Example placeholder, to be replaced only after hardware verification:
EMMC_DEV="/dev/mmcblkX"
```

Back up the **whole** eMMC to a different disk before writing anything. Once the exact device, all unmounted partitions, boot ROM/loader behavior, and image compatibility are verified, the full-disk flashing form on Linux is:

```bash
# DESTRUCTIVE: run ONLY after verifying EMMC_DEV and IMAGE.
# xzcat "$IMAGE" | sudo dd of="$EMMC_DEV" bs=4M status=progress conv=fsync
# sync
```

These lines are intentionally commented out. **Do not write `/dev/mmcblkXboot0`, `/dev/mmcblkXboot1`, or any device that is the active system disk.** Do not assume copying the SD image alone makes eMMC bootable: the RK3528 SPL/DDR loader, U-Boot and partition offsets must match the eMMC boot path. Remove the SD card, select eMMC boot, and collect UART logs on the first boot.

## 8. Power on and verify functionality

Once a valid SD/eMMC image boots, validate both **HAOS services** and **board hardware**. A fresh HAOS image may need network access to download Home Assistant Core.

From the host console (not an application container):

```bash
uname -a
cat /proc/device-tree/model 2>/dev/null || true
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
ip -br addr
systemctl --failed
systemctl status docker haos-supervisor --no-pager
journalctl -u haos-supervisor -n 100 --no-pager
docker ps
rauc status
dmesg | grep -Ei 'mmc|emmc|sdhci|dwmmc|ethernet|usb|firmware' | tail -n 100
```

If the HA CLI is available, run `ha os info`, `ha supervisor info`, and `ha core info`. Open `http://homeassistant.local:8123` or `http://<device-ip>:8123` from another machine.

### Acceptance checklist

- [ ] U-Boot/SPL starts reliably from SD and loads the correct RK3528 DTB.
- [ ] Kernel boots without fatal MMC, mount, regulator, clock, or device-tree errors.
- [ ] SD partitions appear correctly; data storage persists across two reboots.
- [ ] Ethernet obtains an IP address; DNS and internet access work.
- [ ] Docker, OS Agent, HAOS Supervisor and Home Assistant Core start; web onboarding loads.
- [ ] USB ports and any required Wi-Fi/Bluetooth interfaces are enumerated and tested.
- [ ] A/B RAUC slot state is valid; **do not test OTA switching until recovery is proven**.
- [ ] eMMC boot is tested with the SD card removed; partitions and persistent data verified.
- [ ] UART boot logs, `dmesg`, `journalctl`, `ha os info` and image checksum are archived.

Record the exact board PCB revision, DDR type, eMMC size/part number, kernel/DTB/bootloader versions, SD/eMMC device paths and any unsupported functions in the test report.

## 9. Publish the actual local branch to GitHub

**Do this from the local port checkout**, not from `haos-r69max-base`. Review all changes and keep build outputs, keys, credentials and generated images out of source control.

```bash
cd /absolute/path/to/your/local/rk3528-r69max-2-16
git branch --show-current
git status --short
git remote -v
git log -1 --oneline

# Only if origin is not already configured:
# git remote add origin https://github.com/kidd8509/ha-r69max-rk3528.git

# Review and stage ONLY intended source/documentation files.
git diff --stat
git add README.md buildroot-external
git diff --cached --stat
git commit -m "port: add R69MAX RK3528 HAOS board support and documentation"
git push -u origin rk3528-r69max-2-16
```

Adjust `git add` to include any additional, reviewed source files required by the port. **Do not force-push.** This repository's initial `main` commit has unrelated minimal history; publishing the port as its own branch avoids replacing `main`.

## References

- [Pinned Home Assistant OS source](https://github.com/home-assistant/operating-system/tree/c7acf719b948b192ee11dd296f0cdb331e33f0c0)
- [Official HAOS image build instructions](https://contributors.home-assistant.io/os/operating-system/build/)
- [Official HAOS image test/install guide](https://contributors.home-assistant.io/os/operating-system/test/)
- [Official host debugging guide](https://contributors.home-assistant.io/os/operating-system/debugging/)
- [HAOS partition and update system](https://contributors.home-assistant.io/os/operating-system/partitions/)
