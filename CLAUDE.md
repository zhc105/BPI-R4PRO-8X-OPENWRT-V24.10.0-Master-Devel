# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a fork of OpenWrt 24.10 customized to build firmware for the **Banana Pi BPI-R4-Pro-8X** board (MediaTek MT7988A SoC, `mediatek/filogic` target, aarch64 / Cortex-A53). The build system, layout, and conventions are stock OpenWrt; only the target/board files and feed selection are project-specific.

**Git remotes** (as of 2026-05-13):
- `origin` = `https://github.com/zhc105/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel.git` — user's fork, contains the Wi-Fi 7 fix on `main`.
- `upstream` = `https://github.com/BPI-SINOVOIP/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel` — BPI's repo, unmaintained (2 commits total, 3 open issues all unanswered). Do not submit PRs there expecting attention.

**Local commits ahead of upstream/main**: 1 (`bb1c30f1` "fix: BPI-R4-Pro-8X Wi-Fi 7 fails to start on eMMC/SD/NAND boot"). See the Wi-Fi 7 section under Known Issues for the root-cause analysis.

Target device definition lives in `target/linux/mediatek/image/filogic.mk` under `Device/bananapi_bpi-r4-pro-8x[-common]`. Board DTS / DT overlays are under `target/linux/mediatek/files-6.6/arch/arm64/boot/dts/mediatek/mt7988a-bananapi-bpi-r4-pro-8x*.{dts,dtso}`. The kernel is 6.6 (`patches-6.6/`, `files-6.6/`, `config-6.6`).

## Build Host

The build environment must be on a case-sensitive filesystem (Linux/BSD/macOS). The path to the repo **must not contain spaces** (Makefile asserts this).

**Host OS — use Ubuntu 22.04 x64.** There is conflicting documentation in-tree:

- Root `README.md` line 2 claims "compile it on Ubuntu 20.04 x64 host".
- `feeds/mtk_openwrt_feed/autobuild/unified/Readme.md` line 5 states "Minimum requirement: Ubuntu 22.04".

Investigation conclusions:

- The root README's 20.04 claim appears to be boilerplate copied from earlier BPI repos and is **not** backed by any 20.04-specific patching. A full grep of `feeds/mtk_openwrt_feed/` finds no host-version compatibility shims, no `check-host.sh`-style downgrade scripts, and no Ubuntu/glibc/gcc/python version-detection code.
- BPI's customization is concentrated in `target/linux/mediatek/` (board DTS for BPI-R4-Pro-**8X** and the `Device/bananapi_bpi-r4-pro-8x*` recipes in `image/filogic.mk`); `mtk_openwrt_feed/` itself only contains upstream MTK content for plain `bpi-r4` / `bpi-r4-pro` / `bpi-r4-lite` — no `8x` references inside the feed.
- 22.04 is what MediaTek's current autobuild flow targets (newer Go / Python / cmake than 20.04 ships in apt), and OpenWrt 24.10 upstream covers it as well. 20.04 reached end of standard support in 2025-04, so apt repositories are getting harder to use for fresh host setup.

A local container image has been pulled for reproducible builds:

```sh
podman images docker.io/library/ubuntu:22.04   # already present (id 714d1a2f39e9)

# Typical build container invocation (bind-mount repo, persistent build cache)
podman run --rm -it \
  -v "$PWD":/src:Z \
  -w /src \
  docker.io/library/ubuntu:22.04 bash
# Inside the container, install OpenWrt's build deps (build-essential, clang,
# flex, bison, g++, gawk, gcc-multilib, gettext, git, libncurses-dev, libssl-dev,
# python3-distutils, rsync, unzip, zlib1g-dev, file, wget) before running make.
```

Do not build as root inside the container — the OpenWrt Makefile refuses to run as uid 0. Create an unprivileged user, or pass `FORCE_UNSAFE_CONFIGURE=1` only when you know what you're doing.

## Build & Common Commands

```sh
# One-time / after pulling: install feed symlinks into package/feeds/
# IMPORTANT: do NOT run `./scripts/feeds update -a` in this repo. All feeds
# (base, luci, packages, routing, telephony, mtk_openwrt_feed, kenzo, small)
# are vendored under feeds/ at known-good revisions, and mtk_openwrt_feed
# carries MediaTek's autobuild scripts/patches that an `update` would clobber.
# Only run `install` to (re)create package/feeds/<feed>/<pkg> symlinks.
./scripts/feeds install -a

# Configure target / packages
make menuconfig            # interactive
make defconfig             # regenerate .config from current selections

# Full build (downloads sources, builds toolchain, then world)
make                       # parallel: `make -j$(nproc)`
make V=s                   # verbose (show commands + stderr); V=sc adds compile commands

# Targeted rebuilds
make target/linux/{clean,compile}            # kernel + DTS
make package/<name>/{clean,compile,install}  # single package; <name> can be a feed package like package/feeds/packages/foo
make tools/<name>/compile  /  make toolchain/<name>/compile

# Diff current .config against defaults (useful for committing config changes)
./scripts/diffconfig.sh > config.diff

# Cleaning (escalating)
make clean        # wipe build_dir/staging_dir/bin for current target — keeps toolchain
make targetclean  # also remove toolchain
make dirclean     # full wipe incl. host tools and tmp
```

Build artifacts land in `bin/targets/mediatek/filogic/`. The BPI-R4-Pro-8X device produces multiple boot-medium artifacts (eMMC, SD, SPI-NAND): `emmc.img.gz`, `sdcard.img.gz`, `snand-factory.bin`, plus `sysupgrade.itb` and a recovery initramfs — see the `ARTIFACTS`/`ARTIFACT/*` rules in `filogic.mk`.

## Architecture / How the Build Is Organized

OpenWrt has a two-phase top-level Makefile (`Makefile`): the first invocation sets up env and includes `include/toplevel.mk`; sub-makes re-enter with `OPENWRT_BUILD=1` and pull in `rules.mk` + `target/`, `package/`, `tools/`, `toolchain/` Makefiles. Build ordering is encoded as stamp-file dependencies (`tools → toolchain → target → package → image`).

Key trees:

- **`tools/`** — host-side build tools (compiled for the build machine).
- **`toolchain/`** — cross-compiler, libc, kernel-headers for the target arch.
- **`target/linux/<target>/`** — board-support: kernel config (`config-<ver>`), DTS (`files-<ver>/.../dts/`), patches (`patches-<ver>/`), subtarget defs (`<subtarget>/target.mk`), image recipes (`image/<subtarget>.mk`), default rootfs overlay (`base-files/`). For this repo, work happens in `target/linux/mediatek/` with subtarget `filogic`.
- **`package/`** — in-tree packages. Each has its own `Makefile` using `include $(INCLUDE_DIR)/package.mk` and `Package/<name>` blocks.
- **`feeds/` + `feeds.conf.default`** — external package collections. **In this repo `feeds/` is vendored** (all eight feeds are checked in at fixed revisions, with MediaTek's `mtk_openwrt_feed/autobuild` patches already applied), so the normal `feeds update` step is skipped. `feeds.conf.default` documents the upstream pins for reference. `./scripts/feeds install -a` is still required after a fresh clone to materialize `package/feeds/<feed>/<pkg>` symlinks.
- **`include/`** — shared make fragments (`package.mk`, `kernel.mk`, `image.mk`, `kernel-defaults.mk`, etc.). When editing package or image recipes, the macros being called are defined here.
- **`scripts/`** — host helper scripts invoked by the build (`feeds`, `getver.sh`, `diffconfig.sh`, image post-processing).
- **`dl/`** — downloaded source tarballs cache (populated by build, do not commit).
- **`config/`, `Config.in`** — top-level Kconfig (`make menuconfig` UI).

### Image recipe model (relevant for board work)

Each device in `target/linux/mediatek/image/filogic.mk` is a `Device/<name>` block:

- `DEVICE_DTS*` selects the device tree and overlays from `files-6.6/.../dts/mediatek/`.
- `DEVICE_PACKAGES` adds device-specific kmods/firmware on top of the subtarget's `DEFAULT_PACKAGES` (see `filogic/target.mk`).
- `ARTIFACT/<file>` pipelines (`mt7988-bl2 | pad-to | mt7988-bl31-uboot | ubinize-image …`) compose final flashable images by chaining commands defined in `include/image-commands.mk` and `image.mk`. Editing artifact layout = edit these pipelines, not a binary tool.
- `TARGET_DEVICES += <name>` is what actually enables the device for builds.

### Kernel patches & DTS

Kernel source comes from upstream; this tree carries it as patches in `target/linux/mediatek/patches-6.6/` and overlay files in `files-6.6/`. To modify the kernel: edit under `build_dir/target-*/linux-*/linux-6.6.*/`, then `make target/linux/update` (regenerates patches) or `make target/linux/refresh`. Do not edit the extracted source in `build_dir/` and expect it to persist — only patches and `files-6.6/` are tracked.

## Conventions to Respect

- Don't `git add` `build_dir/`, `staging_dir/`, `tmp/`, `bin/`. `dl/` and `feeds/` ARE tracked in this repo (vendored) — do not delete or `feeds update` them. `.config` is also typically not committed; commit `diffconfig.sh` output instead if a config change is intended.
- Package and image Makefiles are **OpenWrt-style** — they `include $(INCLUDE_DIR)/package.mk` / `image.mk` and use `define Package/...` / `define Build/...` / `define Device/...` blocks, not plain GNU make rules. Follow patterns in nearby packages rather than inventing structure.
- Custom feeds are pinned by commit in `feeds.conf.default` (note the `^<sha>` syntax). When updating, change the pinned commit, don't leave a feed floating.

## Known Issues / Caveats

- **Upstream issue tracker is unattended.** The GitHub repo `BPI-SINOVOIP/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel` has 3 OPEN issues (#1, #2, #3) and zero responses from maintainers. Don't expect upstream help.
- **`airoha/EthMD32.dm.bin` missing during kernel build — REQUIRES A PRE-STEP** (issue #1, multiple reporters; reproduced 2026-05-12). The bare `make` of a fresh checkout fails with `No rule to make target '../../linux-firmware-20241110//airoha/EthMD32.dm.bin'` because `target/linux/mediatek/filogic/config-6.6` sets `CONFIG_EXTRA_FIRMWARE="airoha/EthMD32.dm.bin airoha/EthMD32.DSP.bin"` and `CONFIG_EXTRA_FIRMWARE_DIR="../../linux-firmware-20241110/"`, but nothing in the standard `make world` flow extracts that firmware tree. The extraction is done by an MTK autobuild hook (`airoha_firmware_install()` in `feeds/mtk_openwrt_feed/autobuild/unified/filogic/rules` line 51) that is *not* run by `make world` — only by MTK's own `autobuild` driver. **Workaround**: before running `make`, prepare the firmware package once:
  ```sh
  make package/firmware/linux-firmware/{clean,prepare} V=s
  ```
  This downloads `linux-firmware-20241110.tar.xz` (386 MB from kernel.org) into `dl/` and extracts it into `build_dir/target-aarch64_cortex-a53_musl/linux-firmware-20241110/`, which is exactly where the kernel's `CONFIG_EXTRA_FIRMWARE_DIR="../../linux-firmware-20241110/"` resolves to (relative to `linux-mediatek_filogic/linux-6.6.93/`). After this, re-run `make -j$(nproc)`.
- **`target/install` (image packaging) is racy under high `-j`** (reproduced 2026-05-12 with `-j16`). After kernel + packages compile successfully, the image-assembly step in `target/linux/mediatek/image/` can fail with a vague `ERROR: target/linux failed to build` and no useful detail in the summary log. Re-running `make target/install V=s -j1` succeeds cleanly. Recommended recovery: when a `-j$(nproc)` world build fails late, before assuming a real bug, run `make target/install V=s -j1` to see if it was just a parallel-install race.

### Official documentation (Banana Pi wiki)

Authoritative sources for board hardware, flashing, and bring-up — these were used to verify the flashing procedure below and override any contradicting info in this tree's `README.md`.

- **Main board page**: https://docs.banana-pi.org/en/BPI-R4_Pro/BananaPi_BPI-R4_Pro — overview, specs, block diagram.
- **Getting Started** (most useful, contains flashing procedure): https://docs.banana-pi.org/en/BPI-R4_Pro/GettingStarted_BPI-R4_Pro — DIP/bootstrap (SW3), serial console settings, SD/eMMC/NAND flashing commands, interface tests, MxL86252C update.
- **Photo gallery**: https://docs.banana-pi.org/en/BPI-R4_Pro/Photo_BPI-R4_Pro — board photos for connector identification.
- **TEST/Power/Thermal**: https://docs.banana-pi.org/en/BPI-R4_Pro/TEST-R4_Pro — power consumption tables (idle ~9 W, with BE14+SFP+fans ~20 W → **12 V / 5 A PSU is required, not optional**), thermal data, and mechanical assembly (4010 fan mandatory on left, BE14 needs heatsink, optional 5020 fan).
- **FAQ**: https://docs.banana-pi.org/en/BPI-R4Pro/BananaPi_BPI-R4Pro_FAQ.
- **Upstream code (this repo)**: https://github.com/BPI-SINOVOIP/BPI-R4PRO-8X-OPENWRT-V24.10.0-Master-Devel — issues list is unattended.
- **OpenWrt mainline PR** for the non-8X BPI-R4-Pro: https://github.com/openwrt/openwrt/pull/21083 — incomplete (missing MXL86282C switch driver, 10G PHY, eth-mux).

Key facts cross-checked from these docs:

- **DIP SW3 boot select** (all jumpers default `1`): SD = `A=1,B=1`; **SPI-NAND = `A=0,B=1`**; **eMMC = `A=1,B=0`**. `system halt!` on console means the selected medium contains no OS.
- **Serial console**: USB-C on **CN41**, 115200 8N1, no parity, no flow control. Power: 12 V on **CN44**.
- **SPI-NAND on R4-Pro-8X is 256 MiB** (DTS `nandflash@10000000 reg=<0 0x10000000>`), not the 128 MB listed in the wiki overview (that's the base R4-Pro variant). MTD partitions: `mtd0=bl2(2M) mtd1=Factory(4M) mtd2=ubi(250M) mtd3=nand(256M whole-chip)`. Factory NAND images are written to **mtd3** (the R4-Pro flashes the whole-chip alias, *not* mtd0 like base R4).
- **eMMC must be flashed from a NAND-booted system**, because SD card and eMMC share one SoC controller (`/dev/mmcblk0` from an SD-booted OpenWrt points at the SD card, not eMMC). Procedure: `dd preloader.bin → /dev/mmcblk0boot0` (after `echo 0 > .../force_ro`), then `dd emmc.img → /dev/mmcblk0`, then `mmc bootpart enable 1 1 /dev/mmcblk0`.
- **eMMC image GPT layout** (parsed from `emmc.img.gz`): `ubootenv 0.5M`, `factory 2M`, `fip 4M`, `recovery 128M`, `production 448M`. Total declared ≈ 608 MiB on 8 GiB eMMC, so **GPT backup is fixed at 608 MiB; the remaining ~7.4 GiB is unallocated**. Reclaim with `sgdisk -e /dev/mmcblk0 && sgdisk -n 0:1245184:0 -t 0:8300 …` then `mkfs.ext4`/`mkfs.btrfs` for a Docker data partition.

### Verified end-to-end build (Ubuntu 22.04 container, 2026-05-12)

The full build succeeded with the sequence above and produced all expected artifacts in `bin/targets/mediatek/filogic/`: eMMC img (187 MB), SD img (189 MB), SPI-NAND factory (201 MB), `squashfs-sysupgrade.itb` (112 MB), `initramfs-recovery.itb` (75 MB), per-medium `bl31-uboot.fip` + `preloader.bin`, `emmc-gpt.bin`, and `kernel-debug.tar.zst`. Total elapsed real time on a 16-core host was roughly tens of minutes (tools+toolchain dominate the first run; subsequent rebuilds are much faster).
- **BPI's official prebuilt image is not reproducible from this HEAD** (issue #1, kolbasky's report): even modules compiled out of this tree may be ABI-incompatible with the image hosted on the BPI product page. Treat this repo as a source-of-truth for *building your own*, not for binary-matching the factory image.
- **No upstream mainline support yet.** OpenWrt main has PR `openwrt/openwrt#21083` for BPI-R4-Pro, but per upstream MTK maintainer `frank-w` it's functionally limited (missing MaxLinear MXL86282C switch driver, 10G PHY driver, and eth-mux). So this BPI tree remains the only practical path to a usable image for the 8X variant at present.
- **Wi-Fi 7 (mt7996e PCIe NIC) fix landed in commit `bb1c30f1`** (origin/main). Initial symptoms on the stock BPI tree: driver attaches, PCIe enumerates `pci 0000:01:00.0: [14c3:7990]`, but every MCU message times out with `Failed to start patch` → `probe ... failed with error -11`. Reading `MTK WED WO Firmware Version: ____000000` in dmesg is a **red herring** — it's just an unfilled metadata field in `mt7988_wo_*.bin` trailer, not a load failure. Root cause was **two latent BPI bugs that compound**:

  1. **`bootconf_extra=` is empty** in all three env files (`bananapi_bpi-r4-pro-8x_{emmc,sdmmc,snand}_env`) inside `package/boot/uboot-mediatek/patches/999-add-bananapi_bpi-r4-pro-8x.patch`. The BPI bootcmd does `bootm $addr#$bootconf#$bootconf_emmc#$bootconf_extra`, so an empty `bootconf_extra` means the `wifi-mt7996a` overlay is never merged into the fdt — mt7996 chip lacks its enable/PCIe-power/GPIO nodes and never responds to MCU messages. BPI shipped NAND with a manually-`saveenv`'d `bootconf_extra=mt7988a-bananapi-bpi-r4-pro-8x-wifi-mt7996a` so factory NAND Wi-Fi 7 works; freshly flashed eMMC/SD does not until env is fixed.

  2. **The wifi overlay references `&i2c_wifi`** but the kernel base DTS (`mt7988a-bananapi-bpi-r4-pro-8x.dts`) exposes that i2c bus under the label **`imux3_wifi`** (and aliases it as `i2c6`). Even after `bootconf_extra` is set correctly, u-boot's `fdt_overlay_apply()` returns `FDT_ERR_NOTFOUND` and the boot falls back into `boot_tftp_forever` looking for a non-existent TFTP server. Fixed by retargeting the overlay's `fragment@1` to `&imux3_wifi`.

  Commit `bb1c30f1` also adds `bananapi,bpi-r4-pro-8x` to the `package/boot/uboot-envtools/files/mediatek_filogic` bpi-r4 case so `/etc/fw_env.config` is auto-generated on first boot (the bpi-r4 case already covers the fitblk + emmc/ubi/default code paths and applies as-is).

  Verified end-to-end on real hardware 2026-05-13: mt7996e completes probe, phy0 registers, and AP comes up on 2.4 GHz (40 MHz) and **6 GHz, channel 37, width 320 MHz** with MLO multi-link. Full reflash (`emmc.img.gz` + `emmc-preloader.bin`) from a clean state Just Works™ with no manual `fw_setenv` required.

  How to re-verify on a future fresh flash: boot eMMC, `iw dev` should show `phy0.0-ap0` (2.4 GHz) and `phy0.2-ap0` (6 GHz 320 MHz). If only `phy0.0-ap0` shows and `dmesg | grep mt7996` has `Failed to start patch`, suspect `bootconf_extra` was reset — check `fw_printenv bootconf_extra` and re-`fw_setenv` if needed (shouldn't happen if the fixed fip+BL2 are flashed, since the new compiled-in default env has the right value).

### Verified end-to-end flashing run (2026-05-12, BPI-R4-Pro-8X-BE14 hardware)

Confirmed the official BPI procedure works in practice, with these concrete details worth remembering:

- BPI shipped NAND pre-flashed with OpenWrt 24.10-SNAPSHOT, so the canonical path is "boot NAND → dd our `emmc-preloader.bin` + `emmc.img.gz` to eMMC → flip DIP to eMMC → reboot".
- The USB-UART bridge on CN41 is a **HOLTEK** chip (`idVendor=04d9, idProduct=b534`) registered via `cdc_acm` as `/dev/ttyACM0` — *not* ttyUSB0 like a CH340/CP2102 board would be. Fedora 22.04+ has `cdc_acm` built in, no extra driver needed.
- BusyBox `dd` in OpenWrt does **not** support `status=progress`; the flag silently makes the command fall through to usage text. Drop it.
- After `dd emmc.img → /dev/mmcblk0`, must also run `mmc bootpart enable 1 1 /dev/mmcblk0` (the `/sbin/mmc` binary is preinstalled in the BPI factory NAND image — good).
- First eMMC boot prints `mount_root: overlay filesystem in /dev/fitrw has not been formatted yet` then automatically `mkfs.f2fs` the rootfs_data area inside the `production` GPT partition. Subsequent boots are quiet.
- After eMMC boot, root overlay is **f2fs on /dev/fitrw, ~270 MiB** (production 448M minus sysupgrade.itb 178M). Docker started in this state automatically falls back to `Storage Driver: vfs` because Docker's default overlay2 cannot stack on top of the existing rootfs overlayfs. Functional but space-inefficient — fixing it requires creating a separate filesystem from the unallocated 7.4 GiB at the end of eMMC and pointing Docker `data-root` there (or using btrfs).

### Verified end-to-end build (Ubuntu 22.04 container, 2026-05-12)
