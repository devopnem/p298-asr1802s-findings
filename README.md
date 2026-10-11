# p298-asr1802s-findings

Primary-source findings captured from a **Notion P298** router (hardware version
`P298_N13`, board codename `KAGU` / `ASR1826s`), powered by an
**ASR Microelectronics ASR1802S** SoC (ARM Cortex-A7, Marvell-MMP-derived).

## Why "P298" and not "R291"

This device is also sold as the **Ooredoo R291 V2**, and that string appears in
its own firmware (`EXTERNAL_VERSION = Ooredoo-R291-V2`, inside
`firmware-backup/rootfs-a.bin` at `/etc/default.prop`). It is nonetheless the
wrong name to use for the hardware, for two reasons:

1. **It is ambiguous.** A completely unrelated product — **MediaTek MT7621A,
   MIPS, Linux 3.10.108** — is also sold as a "Notion R291". Searching the
   model string finds both.
2. **The hardware does not use it.** `P298` is what the board actually carries:
   the silkscreen reads `hardware version P298_N13`, the U-Boot banner reads
   `ASR KAGU EVB P298`, ASR's own kernel build string is
   `BB14.07_asr1826sP298_rls3188_...`, and the device's TR-069 model is
   `P298M22I_Ooredoo_Tuni_R2921_1G` — `P298` first.

Both firmware images share the same base software, which is how the R291 name
came to be associated with this board. **P298 is the platform name** used in
this repository, in the DTS (`asr,asr1826s-p298`) and in the workspace repo.
The string `R291` still appears inside the raw dumps because it is genuinely
present in the device's own firmware; that is preserved as evidence, not
endorsed as the product name.

## Contents

This repository contains **data actually read out of the device** (kernel probes,
full NAND dumps, factory firmware backups, PCB photos, chip markings), plus the
**reproducible flash-tooling configuration files** used on it — nothing else:
no patches, no porting work, no narrative analysis. It is meant as a citable
reference for developers working on this platform (Linux mainlining, OpenWrt,
bootloader work).

Narrative analysis and the porting effort live in the main workspace repository,
not here.

## Device identity (as observed from the device itself)

| Property | Value | Source |
|---|---|---|
| Board model string | `ASR 1826s(KAGU) Board EVB` | `probes/dmesg.txt` |
| SoC | ASR Microelectronics ASR1802S (ARMv7 Cortex-A7) | `probes/dmesg.txt` |
| Hardware version | `P298_N13` | `board/chip-markings.txt` (sticker) |
| Kernel (stock) | `3.10.33`, `#1 PREEMPT Thu Apr 11 18:29:34 CST 2024`, gcc 4.8.3 OpenWrt/Linaro 2014.04 | `probes/dmesg.txt` |
| Machine | `ASR ASR1802S (Device Tree Support)` | `probes/dmesg.txt` |
| RAM | 128 MB (`mem=128M@0x0`), CP reserved `cpmem=32M@0x6000000` | `probes/dmesg.txt` |
| Storage | SPI-NAND, 25 MTD partitions | `probes/proc_partitions.txt`, `bootloader/mtd-partitions.txt` |

## Repository layout

```
probes/                     Raw text probes captured over SSH from the live unit
├── dmesg.txt               Full kernel dmesg (boot to steady state)
├── kernel_cmdline.txt      /proc/cmdline
├── bootargs.txt            U-Boot bootargs as parsed by the system
├── mtd_env_strings.txt     U-Boot environment strings (fw_printenv output + env area scan)
├── modem_wifi_dmesg.txt    Modem / Wi-Fi related dmesg excerpt
├── proc_iomem.txt          /proc/iomem (register map)
├── proc_partitions.txt     /proc/partitions (MTD block sizes)
├── proc_interrupts.txt     /proc/interrupts
├── dev_nodes.txt           /dev listing (tty / mtd / cdc data)
├── net_mac_sources.txt     Network interface list
└── live_ssh_session/
    ├── dump.txt            Live SSH session: full output (clock tree, clk_summary,
    │                       netdevs, dmesg excerpt, ...)
    └── registers_and_dt.txt  MMIO register reads + DT clock node + kallsyms clock symbols

board/
├── chip-markings.txt       Silkscreen / marking text read from the physical PCBs
└── pictures/               Photographs of the PCBs

nand-dumps/                 Full SPI-NAND images, one file per MTD partition (mtd0..mtd24)
├── MANIFEST.sha256         SHA-256 checksums (verify before use)
└── mtd0.bin ... mtd24.bin

firmware-backup/            Factory firmware read back from the unit (system slot A)
├── kernel-a.bin            zImage (stock 3.10.33) with vendor DTB appended
├── rootfs-a.bin            SquashFS root filesystem
├── overlay.tar.gz          Root fs overlay
├── etc_config.tar.gz       /etc/config tree
├── oem_data.tar.gz         OEM data partition
├── env_strings.txt         Strings found inside the NVM/calibration partitions
└── vendor_dtb_extracted/
    ├── vendor.dtb          Vendor DTB blob (appended to kernel-a.bin at offset 0x2b9ef0)
    └── vendor.dts          Decompiled vendor DTB

bootloader/
└── mtd-partitions.txt      MTD partition table as reported by the running system

tooling/                    Flash tooling: format spec, our configs, tool inventory
├── README.md               How to run the configs; no vendor binaries are redistributed
├── blf/
│   ├── swdownloader-blf-spec.md   Marvell/ASR `.blf` + `.cfg` format reference,
│   │                             recovered from the tool binary and verified
│   ├── flash-map.md               Verified 128 MiB SPI-NAND address map
│   ├── flash_recovery.blf         Write config: OBM + DTIM A/B + U-Boot A
│   ├── flash_read_mtd0.cfg        Read-only: 256 KiB at 0x0 (the OBM)
│   └── flash_read_all.cfg         Read-only: all 25 partitions, 128 MiB
└── flashing-tools/         One README per vendor tool - name, version, build date,
    ├── README.md           file sizes, md5 checksums, and why none of them can
    ├── Marvell_SWDownloader_v4.8.8.2/README.md      write this board. Binaries excluded.
    ├── Marvell_SwUpgrade_P_V1.05/README.md
    ├── Marvell_SwUpgrade_C_v1.03/README.md
    └── ONTIM_Marvell_MultiDL_v1.4.3.72/README.md
```

### About `tooling/`

The vendor flashing tools are proprietary Marvell/ASR binaries and are **not
redistributed here**. `tooling/flashing-tools/` records only the exact product
name, version, build date, file sizes and md5 checksums so the same tool can be
identified and verified, plus the conclusions reached about each.

`tooling/blf/` contains our own configuration files. `flash_recovery.blf`
parses cleanly and passes the vendor toolchain's own offline validator; the
write it performs is blocked by a missing vendor binary, documented in
`swdownloader-blf-spec.md` section 6. The two `.cfg` files drive the
**read-only** flash-dump path, which needs no vendor blobs and is the way to
establish what is actually on the chip.

All of it runs under `LC_ALL=C LANG=C`; the 2012-2015 binaries abort otherwise.

## Verification

- `nand-dumps/MANIFEST.sha256` lets you verify the NAND dumps are unmodified.
- The DTB inside `firmware-backup/kernel-a.bin` is located at offset `0x2b9ef0`
  (magic `d0 0d fe ed`, totalsize `0x9430`).
- All probes were captured on 2026-08-02/2026-08-04 from the live unit
  (kernel build `Thu Apr 11 18:29:34 CST 2024`).

## Sanitization note

Device-specific identifiers (IMEI, serial numbers, MAC addresses, passwords) have been
replaced with `[REDACTED]` in the **text files**. The binary NAND dumps and firmware
backups are shipped as read from the device, so they may contain the unit's own
NVM/calibration data; treat them as private unit data if you flash or redistribute them.

## Useful cross-references (external work)

- Main workspace repository (porting effort, narrative analysis):
  https://github.com/devopnem/notion-p298
- Linux RFC series for ASR1802S: https://github.com/devopnem/linux-asr1802s-rfc
- OpenWrt port: https://github.com/openwrt/openwrt/pull/24614 (branch `asr1802s-kagu-wip`)