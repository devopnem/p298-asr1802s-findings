# Flash tooling — Marvell / ASR downloader family

This directory documents the vendor flashing tools used on this platform and
ships our own reproducible configuration files for them.

**No vendor binaries are redistributed here.** Each subdirectory under
`flashing-tools/` contains only the exact product name, version, build date,
file sizes and checksums, so that anyone can identify the same tool and
verify they obtained the same files. The tools themselves are proprietary
Marvell/ASR binaries distributed via the vendor and third-party mirrors.

---

## What ships here

| Path | What it is |
|---|---|
| `blf/swdownloader-blf-spec.md` | The `.blf` configuration format, recovered from the tool binary — a reference spec |
| `blf/flash-map.md` | Verified SPI-NAND address map for this board |
| `blf/flash_recovery.blf` | Validated write config: OBM + DTIM-A + DTIM-B + U-Boot A (~1.4 MiB) |
| `blf/flash_read_mtd0.cfg` | Read-only config: 256 KiB at `0x0` (the OBM) |
| `blf/flash_read_all.cfg` | Read-only config: all 25 MTD partitions (128 MiB) |
| `flashing-tools/` | One subdirectory per product, README only |

## Why the config files are here and not in the main workspace

They are the smallest complete description of *what we tried to write to this
device* and *what we asked the device to give back*. They are the part of the
recovery effort that is reproducible and verifiable, so they belong with the
primary-source data. The narrative analysis of the boot chain, the DKB blocker
and the tool comparison lives in the main workspace repository.

---

## The tools

Four products, one shared core. All of them embed or ship the same Marvell
library (`IFL_SWD`), so they share one `.blf` grammar and one set of
limitations. See `flashing-tools/README.md` for the comparison.

The only one with a **native Linux binary** is `Marvell_SWDownloader_v4.8.8.2`
(`swdl_linux`), which is what all work here was done with.

---

## Running the configs

All binaries in this tool family are from 2012–2015 and abort with
`locale::facet::_S_create_c_locale name not valid` unless run under a C locale:

```bash
export LC_ALL=C LANG=C
```

### Offline validation (no router, no USB, no root)

The toolchain ships Linux utilities that run the same parser:

```bash
cd Marvell_SWDownloader_v4.8.8.2
cp <path>/tooling/blf/flash_recovery.blf .
cp <mtd0.bin> <mtd7.bin> <mtd8.bin> <mtd11.bin> .    # from ../nand-dumps/

LC_ALL=C LANG=C ./Utilities/TimBuilder/TimBuilder -r flash_recovery.blf
```

Expected on success:

```
NEZHA platform type
Number of images in blf file is 4
GenerateTim  Successfully.....
TimBuilder  Successfully.....
```

Note: the `.blf` and the image files are resolved **relative to the
executor's directory**, not the current working directory. Keep them next to
the binary.

### Read flash from the device (read-only)

```bash
cd Marvell_SWDownloader_v4.8.8.2
sudo LC_ALL=C LANG=C ./swdl_linux -U flash_read_mtd0.cfg -T 120
```

The device must be in ASR ROM boot mode (USB enumerates as `2ecc:3002`,
product string `KSTR`). Start the tool first, then power-cycle or assert the
`USB_BOOT` test point.

### Write flash (currently blocked — see the spec)

```bash
sudo LC_ALL=C LANG=C ./swdl_linux -D flash_recovery.blf -T 120
```

This parses successfully and then fails with
`OBMImage is NULL in prepareDKB list`. The DKB stage requires four
vendor-internal blobs (`OBMI`, `TIMH`, `WTMI`, `TZII`); `TIMH` and `OBMI` can
be supplied from this repository's dumps, but **`WTMI` and `TZII` are not
part of any MTD partition and are not obtainable**. See
`blf/swdownloader-blf-spec.md` for the full analysis.
