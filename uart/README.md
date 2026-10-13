# UART bring-up — primary-source captures and findings

All captures: FTDI USB-serial, 115200 8N1, no flow control, `/dev/ttyUSB0`.

Captures 1–3 (2026-10-12): **no writes were performed at any point.**

Capture 4 (2026-10-13) differs — see "Writes performed" below. It is the only
capture in which the unit's environment was modified, and the only one that
reached a login prompt.

| file | what it is |
|---|---|
| `boot-oem-to-uboot.log` | first capture via picocom; fully clean, complete chain |
| `boot-second-capture-clean.log` | second capture, own tool; clean, same chain |
| `boot-third-capture-floating-input.log` | third capture; noisy prefix, then clean |
| `boot-fourth-capture-uboot-hang.log` | fourth capture; boots **all the way to a login prompt** (see below) |

## Capture 4: the unit is not bricked, and boots completely

The fourth capture is the most informative one. With the serial adapter attached
and no UART RX wire connected, the unit boots the full chain and reaches a
login prompt:

```
procd: - init -
Please press Enter to activate this console.
```

Pressing Enter yields a root shell. This contradicts the earlier reading that the
unit "hangs" at `get_quiet_from_obm`: **the stock kernel boots, userspace starts,
and a console is offered.** Only the U-Boot *shell* is unreachable — the *system*
is fine.

It also disproves the assumption that the captures stopping at
`get_quiet_from_obm` meant the board was inert. `quiet=1` suppresses the
countdown text and U-Boot's prompt, but it does **not** stop U-Boot from running
`bootcmd` and handing off to the kernel. In a good boot the very next line is:

```
get_quiet_from_obm params->quiet: 1
Uncompressing Linux... done, booting the kernel.
```

So `quiet=1` is a red herring. It is a debug print, not a gate.

Capture 4 also shows the two other UARTs the SoC has, and confirms the PCIe
hardware the port targets:

```
pcie-kagu d4220000.pcie: PCIe0.0: link up
pcie-kagu d4220000.pcie: PCIe1.0: link up
pci 0000:02:00.0: [10ec:f812] type 00 class 0x028000
pci 0000:04:00.0: [10ec:818c] type 00 class 0x028000
```

Two Realtek PCIe radios, both links up at gen 1, with the vendor's
`pcie_rtl8192fr_pwr_on: on=1` / `pcie_rtl8812fr_pwr_on: on=1` asserting the power
GPIOs *before* the links come up. That ordering is consistent with the
`GPIO_ACTIVE_HIGH` polarity this port uses for `reset-gpios`.

## Writes performed (capture 4 only)

From a shell on the booted unit, `fw_setenv` was used to write the U-Boot
environment on **mtd21** only:

```
fw_setenv autostart no
fw_setenv bootdelay 10
fw_setenv bootcmd "echo ===STOPPING UBOOT FOR DEBUG=== ; version"
```

`fw_setenv` is a real write to mtd21, not a RAM-only change (unlike `setenv` at
the U-Boot prompt). mtd21 is the u-boot env partition and is designed to be
rewritten; no other partition was touched. The intended original value is
`bootcmd=mrvlboot`, recoverable from `nand-dumps/mtd21.bin`.

**Unit state after capture 4: does not boot.** U-Boot starts, prints its banner
and `get_quiet_from_obm params->quiet: 1`, then produces no further output.

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

Captures 1–3 stop at exactly the same place:

```
get_quiet_from_obm params->quiet: 1
```

The first picocom capture showed `Terminating...` after this line. Later
captures do not — the board simply produces nothing further. It does not reach
a usable prompt.

**Capture 4 revises this.** With the adapter attached the same stop point is
followed by a complete boot to a login prompt. The boot was never broken; what
was broken was the *ability to observe and interact with it*, and captures 1–3
mistook silence for a hang.

## The NAND dumps are from a different firmware generation

This is the most consequential finding from capture 4, and it bounds what the
existing backups can be used for.

The U-Boot that actually **runs** on the unit is not present in any partition we
hold:

| source | version string |
|---|---|
| unit, running (captures 1–4) | `U-Boot 2014.01-g301f02458 (May 26 2026 - 17:50:03)` |
| `nand-dumps/mtd11.bin`, `mtd12.bin` | `U-Boot 2014.01-g5f745ec (Apr 11 2024 - 18:26:02)` |

A search of all 25 partitions (mtd0–mtd24) finds `g301f02458` **nowhere**, and no
partition contains any May 2026 build date. `g5f745ec` appears only in mtd11 and
mtd12.

Consequences:

- **The running bootloader cannot be inspected offline.** Every conclusion
  about U-Boot behaviour drawn from `mtd11.bin` describes the April 2024 build,
  not the one on the unit. This is why the hang caused by the new `bootcmd`
  cannot be explained — the relevant code is not available to read.
- **`mtd21.bin` is not necessarily this unit's current environment.** It is a
  valid 20-variable env block from the older generation. Its variables are
  readable and it is the best available reference for the original `bootcmd`,
  but it must not be described as a byte-exact backup of this unit's env.
- Reading the real `mtd11` off the unit would resolve both, and requires either
  SWDownloader or an external flash programmer.

This also revises the earlier note that the U-Boot build date matches the
firmware version string `20260526`: that match holds for the **running** image,
which the dumps do not contain.

## Only one of the SoC's three UARTs is enabled

From the live device tree (`probes/live-device-tree-2026-10-12.dts`):

| node | address | status |
|---|---|---|
| `uart@d4017000` | `0xd4017000` | **`okay`** — this is `ttyS0`, the working console |
| `uart@d4036000` | `0xd4036000` | `disabled` |
| `uart@d4018000` | `0xd4018000` | `disabled` |

The boot log confirms only one is ever initialised:

```
pxa2xx-uart.0: ttyS0 at MMIO 0xd4017000 (irq = 59) is a UART1
```

This explains why the board's other 3-pin debug header produces no output: the
UART behind it is switched off in the device tree, so no driver programs the
registers or applies the pinmux. It is a real OEM header whose UART the shipping
firmware disables. Both directions are dead, not merely unused.

The silkscreen `GND` label on that header is also unreliable — the pad marked
ground showed no continuity to any other board ground point.

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

### 2. `bootcmd=mrvlboot` runs to completion without returning to a prompt

Even reaching the command loop would not be enough: `bootcmd=mrvlboot` runs
immediately, and the board goes on to boot the kernel rather than offering a
prompt. Both the own tool and `picocom` were used to send commands (`echo`,
`printenv`, `version`) once U-Boot had been up for over a minute. Nothing came
back.

**Revised by capture 4:** `fw_setenv autostart no` and `fw_setenv bootdelay 10`
*were* applied successfully, and the unit then booted the stock kernel normally.
That is exactly the behaviour of a silent countdown — `quiet=1` hides the
"Hit any key" text, so the 10 s wait is invisible — but it does **not** stop
U-Boot from proceeding afterwards.

So the earlier conclusion "the env is effectively locked" is **wrong**. The shell
needs `autostart=no`, `fw_setenv` can set it from a booted system, and a booted
system was reachable. The barrier was not the environment.

### What capture 4 leaves unexplained

After `fw_setenv bootcmd "echo ===STOPPING UBOOT FOR DEBUG=== ; version"` and a
reboot, U-Boot starts, prints its banner and `get_quiet_from_obm params->quiet: 1`,
and then produces **nothing further** — no `===STOPPING UBOOT FOR DEBUG===`, no
`version` output, no prompt. The unit does not boot.

Commands typed blindly at that point (`setenv bootcmd mrvlboot`, `saveenv`,
`reset`, `mrvlboot`) produced no observable effect, and key-spamming through a
full power cycle did not help.

Since the hang occurs *before* `bootcmd` produces any output, the failure is in
executing the command string, not in the commands it contains — both `echo` and
`version` are present and benign in the April 2024 binary. **The cause cannot be
determined from the dumps**, because the running May 2026 build is not among them
(see "The NAND dumps are from a different firmware generation").

What was ruled out along the way:

- UART RX/TX pads shorted together — no continuity between them
- The console TX path — output is received perfectly, so GND and TX are sound
- The serial adapter itself — TX/RX loopback echoes correctly
- An unidentified test point — a separate net (650 Ω to ground, held low when
  powered), not a UART pin

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