# UART bring-up — findings

First capture: `boot-oem-to-uboot.log`, 115200 8N1 on `/dev/ttyUSB0`.

## The unit is not bricked

The full chain runs to U-Boot:

```
Finish loading TIM from: 0x0000001C
Init DDR: PASS
Verify Image: PASS
SWDownloader 4.9.0.5 for NezhaC
OBM 4.9.0.0 for NezhaC
SPI-NAND: F35UQA001G is found in table
SPI-NAND type mfr_id: cd, dev_id: 6161
U-Boot 2014.01-g301f02458 (May 26 2026 - 17:50:03)
ASR KAGU EVB P298
SoC:   PXA 88AP1829-A0
DRAM:  128 MiB
```

DDR initialises, the SPI-NAND is identified, and U-Boot starts. Whatever the
fault is, it is **not** a dead bootloader, dead RAM, or a dead flash.

## The fault

U-Boot prints its banner and then:

```
get_quiet_from_obm params->quiet: 1
Terminating...
```

Nothing after that. So U-Boot starts, OBM asks it to be quiet, and the boot
command then fails or is absent. This is consistent with a bad `bootcmd`, a
bad `bootargs`, a kernel slot that does not contain a bootable image, or a
load address that no longer holds one.

`get_quiet_from_obm params->quiet: 1` matters for reading the log: OBM sets
`quiet=1`, so U-Boot suppresses most output. Anything after that line would be
invisible unless `quiet` is cleared at the prompt.

## The flash map is confirmed by a second, independent source

The OBM prints its own partition addresses at runtime. They match
`findings/tooling/blf/flash-map.md` exactly:

| OBM output | our map | contents |
|---|---|---|
| `Rsa_Flash_Addr 0x05fc0000` | mtd20 @ `0x5FC0000` | rsa / secure-boot keys |
| `Rsa_Erase_size 0x00040000` | mtd20 = 256 KiB = `0x40000` | same |
| `ASR Flag partition have not been erased` | mtd19 @ `0x5FC...`-1, `0x5F00000` | `asr_flag`, A/B slot indicator |
| `Obm_Flash_Addr 0x00020000` | inside mtd0 (256 KiB) | second OBM copy at +128 KiB |

That map came from NAND dumps; this came from the running first-stage
bootloader. Two unrelated methods, same answer — `mtd19` and `mtd20` are now
solid, which matters because those are two of the three partitions that must
never be written.

## SoC identification

`SoC: PXA 88AP1829-A0` is U-Boot's own naming. `88AP1829` is ASR's part
number for the ASR1826, which sits on the Marvell PXA1829/PXA1826 line. This is
direct support for the target's premise that the part continues Marvell's
PXA18xx family and reuses `ARCH_MMP` / `mach-mmp` and its `mrvl,mmp-*` drivers.

## Naming

`SWDownloader 4.9.0.5 for NezhaC` and `OBM 4.9.0.0 for NezhaC` — "NezhaC" is
ASR's internal codename for this platform, alongside the "KAGU" board code.
Neither appears in the firmware version string, which is
`BB14.07_asr1826sP298_rls3188_1.063.014_20260526_18_00_bld522`.

`U-Boot 2014.01-g301f02458 (May 26 2026 - 17:50:03)` matches the `20260526`
date in that version string and the U-Boot binary in mtd11/mtd12.

`rls3188` is still unidentified. Nothing in this log mentions it.

## Harmless noise

```
OOPS: No right DDR PID found
undefined M26H_LED_ON
```

The first is a DDR SPD-table lookup miss that does not stop init — the MR8/MR5
reads and the 128 MiB report follow it. The second is a missing LED macro. Both
are cosmetic.

## Next step

Get the U-Boot prompt and read the environment. Interrupting autoboot is
read-only and safe; see `docs/uart/README.md` for the exact sequence.