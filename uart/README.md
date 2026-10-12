# UART bring-up — primary-source captures and findings

All captures: FTDI USB-serial, 115200 8N1, no flow control, `/dev/ttyUSB0`.
Taken 2026-10-12. **No writes were performed at any point.**

| file | what it is |
|---|---|
| `boot-oem-to-uboot.log` | first capture via picocom; fully clean, complete chain |
| `boot-second-capture-clean.log` | second capture, own tool; clean, same chain |
| `boot-third-capture-floating-input.log` | third capture; noisy prefix, then clean |

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

DDR initialises, the SPI-NAND is identified, U-Boot starts. The fault is not a
dead bootloader, RAM, or flash.

## The fault

Every capture stops at exactly the same place:

```
get_quiet_from_obm params->quiet: 1
```

The first picocom capture showed `Terminating...` after this line. Later
captures do not — the board simply produces nothing further. It does not reach
a usable prompt, and it does not boot.

## Two independent reasons the U-Boot shell is unreachable

### 1. `autostart=yes` removes the interrupt window entirely

The U-Boot environment contains `autostart=yes`. In U-Boot 2014.01:

```c
static int autoboot_command_autostart(const char *s)
{
        if (!autostart)
                autoboot_command(s);
        return 1;
}
```

With `autostart` set, `autoboot_command()` is never called, so the
"Hit any key to stop autoboot" countdown never runs and `tstc()` is never
polled. **No keypress can abort the boot, and no duration of key-spamming
helps.** `bootdelay=1` is irrelevant on this unit.

The binary does contain the string `Hit any key to stop autoboot: %2d `, so
`CONFIG_AUTOBOOT_KEYED` is compiled in — the interrupt path exists in the code
but is never reached.

### 2. `bootcmd=mrvlboot` hangs instead of returning to the CLI

Even reaching the command loop would not be enough: `bootcmd=mrvlboot` runs
immediately, produces no output under `quiet=1`, and does not return. Both the
own tool and `picocom` were used to send commands (`echo`, `printenv`,
`version`) once U-Boot had been up for over a minute. Nothing came back.

So the env is effectively locked: the shell needs `autostart=no`, and changing
it with `fw_setenv` needs a booted system.

## OBM download mode exists but was not entered

`mtd0.bin` contains, in string order around the point where captures go quiet:

```
Trusted OBM without ProductBuild
User force to download
Wait for download change timeout to true...
<<<< Production_Mode_Flag =
DlFlag
[Non-Prod]Wait for download...
<<<< booting_mode =
[PROD]Enum USB port
bootMode= isDownload=
```

None of these printed in any capture. The OBM took a path that skips
download-wait entirely: the log goes from `rsa_flashaddr_get ok` straight to
`InitFlag=0x00000000` and `ASR Flag partition have not been erased`.

A `RESTORE_Key` exists in the vendor DTS (`gpios = <0x10 0x31 0x01>`, GPIO 16
pin 49, active low) and is the conventional trigger for a forced OBM download
on this class of board. A trial produced no change in the boot log. That trial
is not conclusive — a **data** USB cable must be connected at power-on for the
port to enumerate, and a charge-only cable would make the attempt look
identical to doing nothing.

## The USB download route is blocked on missing blobs

Re-checked against every dump: `WTMI` and `TZII` appear in **none** of them.
`mtd0.bin` contains none of the four DKB magic tags (`OBMI`, `TIMH`,
`UBOOT_A`, `WTMI`, `TZII`) either — those are host-tool constructs, not OBM
strings. This confirms the blocker recorded in
the workspace repo's `docs/recovery/swdownloader_blf_findings.md`: the syntax is solved, the
missing content is binary and cannot be synthesised.

## SoC identification

`SoC: PXA 88AP1829-A0` is U-Boot's own CPU detection. `88AP1829` is Marvell's
part number for the PXA1802 family and `-A0` is the silicon revision. Four
independent sources agree:

| source | string |
|---|---|
| U-Boot | `PXA 88AP1829-A0` |
| OBM | `for NezhaC` |
| `.blf` | `Processor_Type = PXA1802` → `GetPlatFormType()` returns `NEZHA` |
| vendor DTS | `compatible = "asr,1802s-evb", "asr,1802s"` |

Practical consequence: `Processor_Type = PXA1802` in the blf is **correct** for
this board and must not be "fixed" to a platform name.

## Flash map confirmed by two further sources

The OBM prints its own partition addresses at runtime:

| OBM output | map |
|---|---|
| `Rsa_Flash_Addr 0x05fc0000` | mtd20 @ `0x5FC0000` |
| `Rsa_Erase_size 0x00040000` | mtd20 = 256 KiB |
| `ASR Flag partition have not been erased` | mtd19, `asr_flag` |
| `Obm_Flash_Addr 0x00020000` | second OBM copy at +128 KiB inside mtd0 |

U-Boot's own `mtdparts` (mtd21) matches the dump-derived map on every size.
Three independent methods, same answer — mtd0/mtd7/mtd8/mtd20 are solid, and
those are the partitions that must never be written.

## Both stock `bootargs` sources point at partitions that cannot be a rootfs

| source | root= | that partition actually is |
|---|---|---|
| vendor DTS `chosen` | `/dev/mtdblock5` | mtd5 = `mep-ota`, 128 KiB, empty |
| U-Boot env | `/dev/mtdblock11` | mtd11 = `u-boot-a`, 768 KiB |

Neither is rootfs-a/b (mtd15/mtd16). So `mrvlboot` must construct `root=`
dynamically and the stored value is a stale fallback. Anything that boots a
kernel here has to supply `root=` itself.

## Corruption in the third capture is a floating input, not a baud error

The first ~2 KB of `boot-third-capture-floating-input.log` is dominated by

```
0xff 227    0xfe 222    0xfc 166    0xf8 89    0xb8 40    0xf7 35
```

All high bits set, progressively. That is a floating input: an idle UART line
reads `0xFF`, and noise biased high walks down through the sequence. Nothing
was driving the line. From `Finish loading TIM` onward the capture is clean.

A wrong baud rate would give uniform garbage with no correct fragments, so
this is not a baud problem. Wiring was corrected and the following capture was
clean from the first byte.