# SPI-NAND flash map — Notion R291 (P298_N13 / KAGU, ASR1802S)

Linear 128 MiB map of the SPI-NAND as partitioned by the running stock system.

## Provenance

Derived from `probes/proc_partitions.txt` (captured over SSH from the live
unit). The 25 MTD block sizes sum to exactly 131072 KiB = 128 MiB =
`0x08000000`, which means the partition table is **contiguous with no gaps**,
so every offset below is exact rather than inferred.

Cross-checked against `/etc/fw_env.config` (`erasesize 0x20000`, 128 KiB) and
against `nand-dumps/mtd11.bin` (U-Boot) and `nand-dumps/mtd0.bin` (OBM).

## Table

| mtd | offset | size | contents |
|---|---|---|---|
| 0  | `0x0000000` | 256 KiB   | **OBM** boot loader (Marvell, "One Boot Manager") |
| 1  | `0x0040000` | 128 KiB   | |
| 2  | `0x0060000` | 128 KiB   | |
| 3  | `0x0080000` | 128 KiB   | |
| 4  | `0x00A0000` | 128 KiB   | |
| 5  | `0x00C0000` | 128 KiB   | mep-ota (empty on this unit) |
| 6  | `0x00E0000` | 128 KiB   | mep-ota (empty on this unit) |
| 7  | `0x0100000` | 256 KiB   | **DTIM / TIM slot A** |
| 8  | `0x0140000` | 256 KiB   | **DTIM / TIM slot B** |
| 9  | `0x0180000` | 15360 KiB | |
| 10 | `0x1080000` | 15360 KiB | |
| 11 | `0x1F80000` | 768 KiB   | **u-boot-a** |
| 12 | `0x2040000` | 768 KiB   | **u-boot-b** (byte-identical to 11) |
| 13 | `0x2100000` | 5120 KiB  | kernel-a (raw zImage + appended DTB) |
| 14 | `0x2600000` | 5120 KiB  | kernel-b |
| 15 | `0x2B00000` | 20480 KiB | rootfs-a (squashfs) |
| 16 | `0x3F00000` | 20480 KiB | rootfs-b |
| 17 | `0x5300000` | 6144 KiB  | |
| 18 | `0x5900000` | 6144 KiB  | |
| 19 | `0x5F00000` | 768 KiB   | `asr_flag` (A/B slot indicator) |
| 20 | `0x5FC0000` | 256 KiB   | rsa (secure-boot keys) |
| 21 | `0x6000000` | 256 KiB   | U-Boot environment |
| 22 | `0x6040000` | 256 KiB   | |
| 23 | `0x6080000` | 25344 KiB | |
| 24 | `0x7940000` | 6912 KiB  | `MRVL_BBM` (bad-block map) |
|    | `0x8000000` |           | end — total 128 MiB |

## Slot identity — a factory unit has identical A and B

Verified from `nand-dumps/MANIFEST.sha256`. Within one unit:

| pair | identical? |
|---|---|
| `mtd1` == `mtd3` | yes |
| `mtd2` == `mtd4` | yes |
| `mtd5`, `mtd6` | both **zero length** (erased in the factory) |
| `mtd9` == `mtd10` | yes |
| `mtd11` == `mtd12` (u-boot-a/b) | yes |
| `mtd13` == `mtd14` (kernel-a/b) | yes |
| `mtd15` == `mtd16` (rootfs-a/b) | yes |
| `mtd17` == `mtd18` | yes |
| `mtd7` vs `mtd8` (DTIM A/B) | **differ** |
| `mtd19`–`mtd24` | all distinct |

So the factory writes the same image to both slots; only boot-time slot
selection differs. The same holds on a second physical unit for the pairs it
provides.

Consequence: a live unit reporting `system=a` tells you nothing about whether
the B slot is valid — they are the same bytes. A boot failure is therefore not
a "bad slot" problem unless something has already been written to one of them.

## Notes

* **Do not confuse mtd offsets with byte offsets into a raw image.** The
  figures above are byte offsets from the start of the SPI-NAND and, because
  the layout is contiguous, are also the offsets into
  `cat nand-dumps/mtd{0..24}.bin` in mtd order.
* A draft `.blf` produced earlier in this work placed U-Boot at
  `0x000C0000`. That lands **inside mtd7 (DTIM)** and would destroy the TIM
  tables. The correct value is `0x01F80000`.
* `mtd24` (`MRVL_BBM`) is the driver's bad-block map. If the NAND chip has
  factory bad blocks, the MTD layer skips them and a linear concatenation of
  the partitions will **not** line up with raw physical offsets. This must be
  resolved before programming the chip directly with a programmer.
* A/B slot selection does not come from a U-Boot `system` variable;
  `fw_printenv` on the live unit showed no `system`, while the running kernel
  command line did contain `system=a` / `system=b`. The indicator is believed
  to live in `asr_flag` (mtd19) but this is unverified.

## Boot chain as read from the dumps

```
SoC boot ROM
  -> mtd0   OBM              reads DTIM (mtd7/8), verifies RSA signature
                             against the keys in mtd20, LZMA-decompresses
                             OSLO, loads U-Boot
  -> mtd11/12  U-Boot        verifies SHA-256 of each image against the table
                             in DTIM, honours A/B + rollback
  -> mtd13/14  kernel         raw zImage with DTB appended (not a uImage)
  -> mtd15/16  rootfs
```

Consequence for any custom kernel: writing mtd13/mtd15 without also updating
the TIM hashes in mtd7/8 makes the boot loader reject the image and fall back
to the other slot.
