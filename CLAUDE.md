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

**Working invocation on this host (rootless podman, verified 2026-08-04)**:

```sh
podman run --rm --userns=keep-id -v "$PWD":/src:Z -w /src owrt-build:22.04 bash -c 'make -j$(nproc)'
```

`owrt-build:22.04` is ubuntu:22.04 with the deps above **plus `python3-setuptools` and `swig`** (u-boot's prereq check fails without them — the list above is incomplete). `--userns=keep-id` is required, otherwise the container's uid 1000 cannot write this uid-1000 tree and the build dies in `prepare-tmpinfo`. Always `cd` to the repo root first: a stale absolute `cd` makes `-v "$PWD":/src` mount the wrong directory and the build silently no-ops. After a `.config` change run `make target/linux/clean` first; after touching `target/linux/*/base-files/` run `make package/base-files/clean`.

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

## Field Notes (read these BEFORE re-investigating anything)

Two large working documents live in the repo root. They are **excluded from git via `.git/info/exclude`**, so they exist only on this machine and never appear in a fresh clone — but they are the authoritative record of everything done on the running hardware:

- **`RUNTIME-CHANGES.md`** (~79 KB) — every persistent change made to the live BPI-R4-Pro-8X: eMMC repartition, Docker on btrfs, dropbear/firewall hardening, Wi-Fi channel fixes, the `/sbin/smp-mt76.sh` patch (§10l), and the full PPE/flowtable NAT bug investigation (§10c/§10m) with pcap evidence. It also lists which files are persistently modified outside of packages — read it before any sysupgrade, or those changes get silently reverted.
- **`WIFI-DEBUG-NOTES.md`** — Wi-Fi 7 / mt7996 debugging record.

**Flashing**: `sysupgrade` did not work on this board until 2026-08-04 — `bananapi,bpi-r4-pro-8x` was missing from `target/linux/mediatek/filogic/base-files/lib/upgrade/platform.sh`, so stage2 fell through to `nand_do_upgrade` and tried to write the SPI-NAND UBI on an eMMC-booted box (`-28 ENOSPC`, silent reboot into the old firmware). The board is now listed in all three case blocks there. Full procedure, failure diagnosis via `/sys/fs/pstore/console-ramoops-0`, and the "did the flash actually happen" checks are in `RUNTIME-CHANGES.md` §11. Never flash `emmc.img.gz`/`sdcard.img.gz` on this box — they rewrite the GPT and destroy the btrfs data partition (`/dev/mmcblk0p6`); `sysupgrade` only touches `production` (`/dev/mmcblk0p5`).

Anything learned on the hardware should be appended to `RUNTIME-CHANGES.md`, not just left in a chat session — session transcripts are auto-deleted after 30 days.

## Hardware NAT (mtkhnat) — the 10G path

Enabled 2026-08-04. `CONFIG_PACKAGE_kmod-mediatek_hnat=y` (needs `CONFIG_MEDIATEK_NETSYS_V3=y` in `filogic/config-6.6`) plus `&hnat { … status = "okay"; }` in the board DTS. Two driver assumptions had to be patched before the WAN→LAN direction would accelerate: hnat classifies netdevs by **name prefix** (so `mtketh-lan2` must be `"mxl_lan"`, the MXL user-port prefix — not the conduit `eth2`), and `hnat_dsa_fill_stag()` only knew `DSA_TAG_PROTO_8021Q` while this board's switch is `DSA_TAG_PROTO_MXL862_8021Q`. Result: download 6 Gbps at 100 % CPU → **9.2 Gbps at ~0 % CPU**. Full analysis in `RUNTIME-CHANGES.md` §12.

The module is **not autoloaded**; the image ships `hnat-on`, `hnat-off`, `hnat-status`, `hnat-persist`. Note that building HNAT compiles `mtk_ppe_init/start/stop` out of `mtk_eth_soc.c`, so the mainline nft *hardware* flowtable path is gone in such an image; the software flowtable is unaffected but is known-broken here (see `RUNTIME-CHANGES.md` §10m/§13d) and should stay off.

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

### Optional feature: Full-Cone NAT (nft-fullcone), added 2026-05-13

Linux's default `MASQUERADE` is symmetric NAT (RFC3489 NAT type 4 / "Strict"). For P2P / game console workloads, full-cone is much friendlier. This fork ships the immortalwrt 24.10 nft-fullcone patch set (commit `0f2bda64` on branch `feat/nft-fullcone`), which is OFF by default — there is a kmod build target but no user-visible behavior change unless explicitly enabled.

**What was added** (see commit message for full detail):

- `package/libs/libnftnl/patches/001-libnftnl-add-fullcone-expression-support.patch`
- `package/network/utils/nftables/patches/{001-drop-useless-file,002-…-fullcone-…}.patch` — note the `001-drop-useless-file.patch` is a prerequisite that removes `tests/shell/run-tests.sh.rej` shipped (accidentally) inside the nftables 1.1.1 release tarball; without it OpenWrt's quilt wrapper aborts every `prepare`.
- `package/network/config/firewall4/patches/001-firewall4-add-support-for-fullcone-nat.patch` — adds UCI `option fullcone '1'` and the `zone-fullcone.uc` template.
- `package/network/utils/fullconenat-nft/` — out-of-tree kmod pulling `github.com/fullcone-nat-nftables/nft-fullcone @ 07d93b62` (the upstream repo is **archived**, last push 2023-05-17 — no fixes coming, the 010 patch in this package is the only kernel-6.6 adapter).

**Version alignment with immortalwrt is exact** (libnftnl 1.2.8, nftables 1.1.1, firewall4 18fc0ead all match commits byte-for-byte). Zero hunk reject. If you bump any of these three components, re-check the patches don't go stale.

**How to enable**:

```sh
echo CONFIG_PACKAGE_kmod-nft-fullcone=y >> .config
make defconfig
make -j$(nproc)
make target/install V=s -j1   # if -j$(nproc) hits the install race
# Then on the device:
uci set firewall.@zone[1].fullcone='1'   # @zone[1] is typically wan, verify with `uci show firewall`
uci commit firewall
fw4 reload
nft list ruleset | grep fullcone         # should now show fullcone statements
```

**Caveats**:

- **UDP only**. TCP falls back to standard MASQUERADE (still symmetric). Enough to flip game-console NAT-type tests to "Open" / NAT1 (those probe UDP via STUN). TCP-based P2P apps see no improvement.
- **Interaction with MTK private HNAT — theory says they cooperate, runtime not yet measured.** Source-level analysis of `drivers/net/ethernet/mediatek/mtk_hnat/` (added by `mtk_openwrt_feed` patch 999-2745) shows the HNAT driver is **NAT-type agnostic**:
  - HNAT itself does NOT allocate ports. In `hnat_nf_hook.c` the FOE entry's `new_sport` is filled with `ntohs(pptr->src)` straight from the skb after netfilter has finished its work — HNAT is a transparent cache for whatever conntrack decided.
  - Hook priorities give all NAT decisions room to run: `NF_INET_PRE_ROUTING` at `NF_IP_PRI_FIRST + 1` (= -32767, fast-path lookup only — on miss it lets the packet continue through conntrack at -200, DNAT at -100, etc.), and `NF_INET_POST_ROUTING` at `NF_IP_PRI_LAST` (= INT_MAX, fires after SNAT at +100 and the fullcone expression which sits in the same NAT chain).
  - Therefore the expected behavior after enabling fullcone is: the first egress packet of a new flow traverses the full netfilter stack, `nft_fullcone_*_eval` writes a port-consistent SNAT, then HNAT POST observes the post-fullcone 5-tuple and caches it into the PPE FOE table. Subsequent egress packets get hardware fast-forwarded **while preserving the fullcone port mapping**. For unsolicited inbound, HNAT PRE misses (no FOE entry yet for the reverse direction), the packet falls through to conntrack and the fullcone PREROUTING handler, which looks up the reverse mapping and creates a new conntrack; HNAT POST then caches the reverse flow. **No code-level path makes HNAT bypass or override fullcone's decision.**
  - The remaining unknowns are unrelated to hook ordering: (a) whether the `/sbin/mtkhnat` userspace daemon's conntrack scanning interacts with fullcone's private `nat_mapping` table, and (b) whether HQoS / sp_tag / vlan handling in the FOE entry layout makes any assumption that breaks for fullcone-mapped reverse flows. Both can only be answered by running real traffic.
  - **Suggested verification methodology (still two-step, not because cooperation is expected to fail but to make any divergence easy to localize):**
    ```sh
    # Step 1: characterize fullcone in isolation on the upstream Linux offload path
    /etc/init.d/mtkhnat stop && /etc/init.d/mtkhnat disable
    uci set firewall.@defaults[0].flow_offloading_hw='1'
    uci commit firewall && /etc/init.d/firewall restart
    # → run STUN test, NAT-type test, BT/game traffic. Expected: fullcone OK.
    # Step 2: re-enable mtkhnat and repeat.
    /etc/init.d/mtkhnat enable && /etc/init.d/mtkhnat start
    # → repeat tests. Expected (per code analysis): fullcone still OK, throughput jumps to ~10 Gbps line rate.
    # If step 2 regresses, the divergence isolates to mtkhnat userspace or HQoS behavior, not the kernel hook ordering.
    ```
  - Performance trade-off if you do end up choosing Linux offload only: MTK HNAT does ~10 Gbps line-rate, Linux `flow_offloading_hw` does ~2.5–4 Gbps on this hardware.
- **NAT classification test commands** (run from a LAN client):
  ```sh
  # quick "what's my NAT" via STUN
  stun stun.l.google.com:19302       # apt install stuntman-client; look at "mapped address"
  ```
  Run twice; if the mapped public port is identical across runs ⇒ fullcone is working. If it differs ⇒ symmetric (something's wrong).

**Why this is a separate branch, not on `main`**: keeps `main` aligned with the BPI hardware-bring-up story (Wi-Fi 7 fix, Docker enablement) and isolates a Chinese-community-only convenience feature. Merge to main once runtime-verified on hardware.

### Verified end-to-end build (Ubuntu 22.04 container, 2026-05-12)
