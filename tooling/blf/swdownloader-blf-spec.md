# `.blf` configuration format — Marvell / ASR SWDownloader family

Reference specification recovered by reverse engineering
`Marvell_SWDownloader_v4.8.8.2/swdl_linux` (ELF32 i386, statically linked,
not stripped, ships DWARF `debug_info`) and confirmed by bisection against the
tool's own parser.

Everything below is a **format description**. The analysis of *why* this
format exists and what blocked us is summarised in the final section and lives
in full in the main workspace repository.

For the internals of the engine itself — USB transport, the WTP opcode table,
the image-preparation pipeline, path resolution — see
**`swdl_linux-internals.md`** in this directory.

---

## 1. File shape

A `.blf` is a **plain-text INI-like file**, line-based, with:

* section headers `[Name]`
* `Key = Value` lines
* `;` comment lines

Despite the name it has nothing to do with CANoe/BLF binary logging.

## 2. Parser behaviour (matters when authoring)

| Behaviour | Detail |
|---|---|
| Section header | `isHeader()` requires the line to start with `[` |
| Key lookup | line-oriented; the key is matched inside the current section only |
| Value extraction | everything after the first `=`, then `str_remove_spaces()` |
| **Comments** | **must all appear before the first `[Section]`** |

### Trap 1 — comments must be in the header block

A `;` comment line *between* sections makes the parser lose section tracking.
The symptom is misleading and appears much later in the flow:

```
complete to parse ExtendedReservedData
IFLSWD exception:
unknown DSAID in ParseDSAInfo
```

Every key validates fine in isolation; only the misplaced comments cause it.

### Trap 2 — `Trusted` must be `0`

With `Trusted = 1` the parser enters `CBlfParser::ParseDSAInfo()` and aborts
with `unknown DSAID in ParseDSAInfo` unless a `DSA_Algorithm` is declared.
That path exists to **re-sign the TIM**. It must not be taken when the DTIM
images are already vendor-signed and are being pushed verbatim.

### Trap 3 — `Processor_Type` takes SoC names, not platform names

`CBlfParser::GetPlatFormType()` compares against a table of **SoC part names**
and prints the *resulting* platform name back to you:

| value in the file | prints | enum |
|---|---|---|
| `PXA688`, `PXA1202`, `PXA2128` | `MMP2 platform type` | 2 |
| **`PXA1802`** | **`NEZHA platform type`** | **6** |
| others (`PXA1908`, `PXA1928`, …) | their own names | 8–15 |

`NEZHA` and `Falcon` are **rejected**. `strings mtd0.bin` contains `Falcon`
and `NezhaC`, but those are the OBM's own build identifiers, not `.blf` tokens.
This board is `PXA1802` (confirmed: `OBM 4.9.0.0 for %s`, `Falcon`,
`NezhaC`, `Falcon A0 spi pinmux` inside `nand-dumps/mtd0.bin`).

### Trap 4 — there is no `FlashType` key

The flash family comes from `[Flash_Properties] Flash_Family`. There is no
`FlashType`, `FlashTypeNum` or `Platform` key anywhere in the parser.

---

## 3. Accepted value tables

### `Flash_Family` — `CBlfParser::GetFlashType()`

Exactly four accepted tokens, case-sensitive:

```
NAND        eMMC        SPI-NOR        SPI-NAND
```

Anything else → `Unknown flash Type to support....`

This board: **`SPI-NAND`**.

### `ImageType`

```
OBM   OBMI   PRIMARY   RECOVERYIMAGE   CPIMAGE   PPSETINGIMAG
CUSTMOZIEDIMAGE_TYPE1 .. CUSTMOZIEDIMAGE_TYPE20
```

(The `CUSTMOZIED` spelling is the tool's own typo.)

### `BLF_Version`

```
V2.0.0   V2.1.0   V2.2.0   V3.0.0
```

Does not change the DKB code path.

### DKB stage image identifiers

`CIFL_SWD::PrepareMMP2DKBlist()` / `PrepareTwoStagDKBlist()` match
`CImageDescription::ImageId()` against these literals:

| literal | meaning |
|---|---|
| `TIMH` | TIM/DTIM header binary |
| `OBMI` | boot-ROM flasher |
| `WTMI` | WTM manager |
| `TZII` | TrustZone initialisation image |

`CImageDescription::ImageId()` stores its string at `this + 0x44`; the setter
is `CImageDescription::ImageId(std::string)`.

---

## 4. Sections and keys

### `[TIM_Configuration]`

```ini
Processor_Type     = PXA1802       ; required, see Trap 3
Number_of_Images   = 4             ; required
BLF_Version        = V3.0.0
FlasherLoadAddress = 0x00000000
Trusted            = 0             ; must be 0, see Trap 2
ProductionMode     = 0
Version            = V3.0.0
Issue_Date         = 2026-10-10
OEM_UniqueID       = NOTION_R291_P298_N13
```

The same section also carries TIM header fields written back as
`Processor Type` / `TIM Configuration` by `CTimDescriptor::TimHeader()`.

### `[Flash_Properties]`

```ini
Flash_Family      = SPI-NAND       ; required, see Trap 4
Flash_Block_Size  = 0x20000        ; from /etc/fw_env.config erasesize
Flash_NandID      = ...
Data_Area_Size    = ...
Spare_Area_Size   = ...
Flash_Options     = 0x0000
Erase_All_Flash   = 0
Reset_BBT         = 0
Skip_Blocks_Number, FBF_Sector_Size, Flash_Family, Flash_NandID,
Max_FBF_Split_Size, Max_OTA_Image_Size, OTA_Image_Split_Size,
Max_Upload_Split_Size, UE_Boot_Option, UE_Options
```

Verified by bisection: **every key except `Flash_Family` may be omitted and the
file still parses.** The geometry keys left out of our configs are
board-specific and unknown, and shipping a guessed NAND part number into a
flash writer is worse than omitting it.

### `[Image_List]`

Per-image keys are constructed by the parser as

```c
snprintf("%d_Image_%s", n, key)      /* n is 1-based */
```

so the on-disk form is `<N>_Image_<Key>`:

```ini
1_Image_Path                = mtd0.bin   ; required
1_Image_Type                = OBM
1_Image_ID_Name             = OBMI      ; feeds ImageId()
1_Image_Image_ID            = OBMI
1_Image_Next_Image_ID       = 2
1_Image_Load_Address        = 0x00000000 ; MANDATORY for the OBM image
1_Image_Flash_Entry_Address = 0x00000000
1_Image_Partition_Number    = 0
1_Image_Erase_Size          = 0x20000
1_Image_Tim_Included        = 1
1_Image_Enable              = 1
```

Accepted sub-keys: `Path`, `Type`, `ID_Name`, `Image_ID`, `Next_Image_ID`,
`Partition_Number`, `Tim_Included`, `Hash_Algorithm_ID`,
`Size_To_CRC_in_bytes`, `Image_Size_To_Hash_in_bytes`, `Enable`,
`Flash_Entry_Address`, `Load_Address`, `Erase_Size`, `ImageNumber`.

Omitting `Load_Address` on the OBM image aborts with:

```
ERROR in Prepare %s DKBlist to be download into BootRom!
NO OBMI in image list or not Appoint FlasherLoadAddress in blf
```

---

## 5. Read-only configuration (`.cfg`)

A **separate** format, used by `swdl_linux -U <file>`. Parsed by
`CUploadCfgParser::ParseUploadCfgFile()` (`swdl_linux` `0x081297d0`–`0x0812a930`;
section and key names from the pointer table at `0x8335b48`–`0x8335b90`).
Per-area keys use `snprintf("%d_%s", n, key)` — no `Image` infix here, and the
index is 0-based in practice.

```ini
[Upload_Property]
Upload_Area_Nums    = 1
Blf_Version         = V2.1.0        ; only V2.0.0 / V2.1.0 / V2.2.0 accepted
PlatForm_Type       = SPI-NAND     ; goes through GetFlashType, same tokens
Flash_Family        = SPI-NAND
Max_Upload_Size     = 0x40000
ResetUE_AfterUpload = 0
Reset_DelayTime     = 0

[Upload_Areas_List]
0_Area_Type                = Flash                  ; must CONTAIN "Flash"
0_Area_SubType             = Default_Flash_Probe    ; must CONTAIN this string
0_Area_Partition           = 0
0_Area_Offset              = 0x00000000
0_Area_DataSize            = 0x00040000
0_Area_UploadWithSpareArea = 0
0_Area_UploadData_SaveFile = mtd0.bin
```

`Area_Type` and `Area_SubType` are validated by
`CUploadCfgParser::FindType()` / `::FindSubType()`, which are just
`std::string::find()` calls for `"Flash"` (5 chars) and
`"Default_Flash_Probe"` (19 chars) — substring, not equality.

### Trap 5 — `Upload_Area_Nums` must be inside the section

It is read as `GetBlfInt("Upload_Property", "Upload_Area_Nums", 0)`. Putting
it **before the first `[Section]`** reads as 0 and the parser aborts with

```
there is no upload area in UploadCfg file, please check
Parse UploadCfg file Failed
```

with no hint that the value was simply never found.

### Trap 6 — the locale, again

```
terminate called after throwing an instance of 'std::runtime_error'
  what():  locale::facet::_S_create_c_locale name not valid
```

Run under `LC_ALL=C LANG=C`. And `swdl_linux` needs root; without it the
binary exits before printing its banner, which looks like a crash rather than
a permission problem.

Parser feedback strings, useful for bisection:

```
Parsing UploadCfg file: %s
there is no upload area in UploadCfg file, please check
BLF version error, please check
ResetDelayTime can't be larger than 255
Number of Area in UploadCfg file is %d
Upload Area %d parameter:  + SubType/Partition/Offset/DataSize/ResetUE/ResetTimeDelay
Upload File name is: %s
Begin to upload data from 0x%x to 0x%x to file %s
Uploading data with Spare Area from 0x%x to 0x%x to file %s
```

**Status: syntax confirmed against the binary; the actual USB transfer has not
yet been exercised against hardware.**

---

## 6. Write path — SOLVED 2026-10-13

> **This section previously concluded the write path was dead. That was wrong.**

The error was a bug in our own `.blf`, not a missing vendor blob.

```
Platform is NEZHA.....
Preparing NEZHA DKBlist
IFLSWD exception: OBMImage is NULL in prepareDKB list
```

### 6a. Root cause: `Image_ID` is numeric

`*_Image_Image_ID` is parsed as an **integer**. Our `flash_recovery.blf` wrote
`1_Image_Image_ID = OBMI`, which parses as `0`, so the DKB builder never
matched the entry and the list came back empty.

```ini
; wrong
1_Image_Image_ID = OBMI
; right — ASCII hex of the tag
1_Image_Image_ID = 0x4F424D49   ; 'O' 'B' 'M' 'I'
```

`*_Image_ID_Name` may stay a plain string. Changing only this key flips
FAIL → OK.

### 6b. `WTMI` and `TZII` are not required

The four literals in §3 sit next to `%s/temp/TZII.bin`, `%s/temp/WTM_h.bin`,
`%s/temp/OBM_h.bin` — all **temp files the tool generates**. Proof: run the tool
on a known-good vendor `.blf` and read its own output.

`DKB_ntimheader.bin` (744 B) holds a single `IBKD` entry and zero `WTMI`/
`TZII`. `Dkb.bin` is the `OBMI` payload wrapped in a 4-byte TIM header — with
our images, 262,148 = 262,144 + 4.

Empirically: `TIMH` alone fails, `OBMI` alone fails, `TIMH` + `OBMI`
succeeds. Two entries, not four.

### 6c. Two more bugs

| what | error | fix |
|---|---|---|
| `Erase_Size` < image file size | `ImageFileSize is larger than ImageEraseSize,ImageID: OBMI` | `0x40000` for 256 KiB images |
| two enabled images sharing `Flash_Entry_Address` | `Images TIMH and OBMI overlap in flash` | distinct addresses |

`Erase_Size` is the region the image occupies, not the NAND block size.

### 6d. Result

`p298_fixed.blf` builds a package from our own `mtd0.bin` + `mtd7.bin`:

```
LC_ALL=C LANG=C ./Utilities/MakeDownloadPackage/MakeDownloadPackage \
    -r p298_fixed.blf -o pkg.zip
# Make Download Package successfully....   exit 0
```

`cfg.ini` returns `PlatFormType = 6` (NEZHA), `FlashFamilyType = 4`
(SPI-NAND) — both correct.

### 6d-bis. HAZARD: the tool overwrites the TIMH input image

**`MakeDownloadPackage` rewrites the `TIMH` image in place**, offline, with no
hardware involved. Reproduced 2026-10-13:

```
before:  mtd7.bin  262144 B  7de63ddc29453b0a...
after:   mtd7.bin    248 B  f4e1b35d7e74e84c...
```

The 248 B file is a freshly generated TIM header (`HMIT`/`IMBO`, vendor
`Sky!`, dated `0x20141006`) — not our dump. `mtd0.bin` (the `OBMI` image) is
**not** touched.

This is exactly the write-through risk §6c of the main doc warned about, but it
fires on the **offline utility**, not only on `-D`. `flashing_tools/
Marvell_SWDownloader_v4.8.8.2/` holds `mtd7.bin` as a **symlink** into
`findings/nand-dumps/` — running the tool there would destroy the
primary-source DTIM dump.

**Rules:**
- Always run in a scratch directory containing **real copies**.
- Never run from `flashing_tools/Marvell_SWDownloader_v4.8.8.2/`.
- `flashing_tools/p298-swd/` is the prepared, safe run directory.

Method: bisected against `DTU-AIR720-MODUL_1.7.2_Luat_V0018_ASR1802.blf`, a
known-good vendor package for the Air720 **module** (`PXA1202` = WUKONG,
SPI-NOR). Flipping it to `PXA1802` + `SPI-NAND` — our board — built cleanly,
which is what located the three bugs.

Verified non-issues, each tested individually: comments between sections,
`BLF_Version` V2.0.0 vs V3.0.0, `FlasherLoadAddress` present or absent,
dropping `[EraseOnly_Option]` / `[Reserved_Data]` /
`[Extended_Reserved_Data]` / `[Flash_Options]` / `[UE_Options]` /
`[BLF_Version]`, `Image_Type` `OBM`/`PRIMARY` instead of `RAW`, index-style
`Next_Image_ID`, omitting `Size_To_CRC_in_bytes`.

### 6e. Still open

No USB transfer has been exercised against hardware. See §8.

---

## 8. USB transport — how the handshake actually works

`swdl_linux` needs **no kernel module**. It opens `/dev/bus/usb` and speaks raw
usbfs, parsing descriptors itself (`End_USB_DEVICE_DESCRIPTOR`,
`End_USB_CONFIG_DESCRIPTOR`, …). No libusb. The `rmmod wtptp.ko` in its output
is cleanup of a Marvell module shipped in some Android images; its absence is
harmless and the tool continues.

### 8a. CLI

```
./swdl_linux -D BLF_filename  [-T Timeout(seconds)] [-N]
./swdl_linux -D BLF_filename -S [-T Timeout(seconds)] [-N]   # single device
./swdl_linux -U Upload_config_filename
```

`-D` also accepts a compressed download package: *"File format error, please
input \".zip\" or \".rar\" compressed file."*

Paths are resolved relative to the **executor's directory**, not the cwd.
Single instance only — *"there is already an swdl_inux is running"*.
Writes `swdl_linux_wtptp_device_detect.log`.

### 8b. It is a handshake, and it is racy

The device appears briefly then drops — `mtd0` has `USB Enum timeout`, and the
tool carries `NoEnumerationResponseTimeOutError`. The sequence is:

```
Please plug your USB device
Sending Preamble...
Getting Version...
Getting type of Device.....
Open an BootRom Device / Open an BootLoader Device
```

**The tool must already be running when the board powers up.** Start the tool
first, then power-cycle. Starting the tool after enumeration has already timed
out is the most likely reason a download appears to miss.

### 8c. Platform identification table

The tool reads the OBM's identity over USB (`Version: %c%c%c%c`,
`Date: %08X`, `Processor: %c%c%c%c`) and matches this embedded table:

| codes | platform |
|---|---|
| `3215` / `12052009` | `WKNG` (WUKONG = `PXA1202`) |
| `3301` `3302` `3211` | `MMP3`, `TVTD` |
| `3304` `3307` `3311` `3308` `3313` | `HELN`, `NZA3`, `HLN2`…`HLN4` |

`NZA3` = Nezha 3. This is our family — `PXA1802` prints `NEZHA platform type`
in `MakeDownloadPackage` and `cfg.ini` reports `PlatFormType = 6`.

### 8d. Device identity — measured, 2026-10-13

The P298 router enumerates in download mode as:

```
2ecc:3002  'KSTR'  'ASR'
```

present for **1.577 seconds** from power-on, after which the OBM's
`USB Enum timeout` expires and it drops.

This is **not** `USB\VID_1286&PID_8130` ("Marvell R WTP: Tools package USB
Driver for Nezha"), which driver databases list. Those are Monahans/PXA-era
IDs from the module lineage; an earlier note here pointed at them and it was
wrong for this board. Observed value wins.

Consequences, all verified:

- The board is **not** `PROD`-gated. It does enumerate. `[PROD]Don't enum USB
  port` did not apply; the behaviour matches `[Non-Prod]Wait for download...`.
- `swdl_linux` 4.8.8.2 contains **no `0x2ECC` instruction immediate**. Every
  occurrence of the byte pair `cc 2e` decodes as a `call` rel32 displacement.
  Decoded count: `0x2ECC` 0 immediates / 3 coincidences, `0x3002` 0 / 16,
  `0x1286` 0 / 41, `0x8130` 0 / 25.
- `swdl_linux` 4.8.8.2 contains **no `0x2ECC` instruction immediate**. Every
  occurrence of the byte pair `cc 2e` decodes as a `call` rel32 displacement.
  Decoded count: `0x2ECC` 0 immediates / 3 coincidences, `0x3002` 0 / 16,
  `0x1286` 0 / 41, `0x8130` 0 / 25.

**That last point does *not* mean 4.8.8.2 rejects the device.** The missing
constant is by design — but the engine still filters, via a VID/PID whitelist.
See §8f, which supersedes the "cause unknown" conclusion above.

`tools/usb-watch.py` polls sysfs at ~50 Hz to catch the 1.58 s window;
`watch -n1 lsusb` cannot.

### 8f. ROOT CAUSE FOUND — there IS a VID/PID whitelist, 2026-10-14

> **This overturns 8e below.** There is no *GUID* filter in the Windows DLLs,
> but the Linux engine **does** filter on VID/PID, via a whitelist table that
> does not contain `0x2ECC`.

`swdl_linux` is not stripped, so this is read straight from `readelf -sW`:

```
CBulkDevNonDrv_Linux::FindAllSupportedDevices  0x0813b9a0   1666 bytes
CBulkDevNonDrv_Linux::ChecHWIds(long,long,long) 0x0813b220     87 bytes
```

`FindAllSupportedDevices` opens every entry under `/dev/bus/usb` and calls
`ChecHWIds(handle, idVendor, idProduct)` — `idVendor` and `idProduct` read
from the device descriptor at **offsets +8 and +10**.

`ChecHWIds` walks a stride-12 array of `{vid, pid, bcdDevice}` and returns 1
only on a match:

```asm
lea  eax,[ecx+0xc]        ; first entry
loop:
  mov edx,[eax]           ; vid
  cmp edx,ebx             ; vs wanted vid
  jne next
  cmp DWORD PTR [ecx+4],esi   ; vs wanted pid
  jne next
  ... bcdDevice compared only if neither side is 0xFFFFFFFF
  mov eax,0x1             ; match
next:
  add eax,0xc
  test edx,edx            ; vid 0 terminates the scan
  je end
```

The array lives at **va `0x083D72E0`, file offset `0x38e2e0`**, reached via
the global pointer at va `0x08337304`. **30 stored entries, 29 reachable** —
the scan loop starts at table+`0xC`, so entry[0] is used only as the
non-empty-table flag and `0x8086:0xE001` can never match. All have
`bcdDevice = 0xFFFFFFFF` (wildcard). See `swdl_linux-internals.md` §4.2.

| VID | PIDs |
|---|---|
| `0x8086` Intel | `0xC001`, `0xD001` (stored but unreachable: `0xE001`) |
| `0x1286` Marvell | `0x8100`–`0x8106`, `0x8111`–`0x8118`, `0x8128`–`0x812A`, `0x8130`, `0x8164`, `0x8165`, `0x8170`, `0x8174`, `0x8175`, `0x8181`, `0x8182`, `0x8189` |

**`0x2ECC` is absent.** That is why every tool silently skips the board: it
enumerates, `ChecHWIds` returns 0, and nothing is opened. It accounts for
every observation — no `wtp*.log`, a 0-byte `swdl_linux_wtptp_device_detect.log`,
and no kernel driver bound.

`0x1286` is the Monahans/Marvell vendor ID from the `wtptp.inf` hardware IDs
(`USB\VID_1286&PID_8130`). The P298's `0x2ECC` is a different, newer vendor ID,
which is the same "generation gap" as the 4.7.2.14 → 4.9.0.5 OBM difference.

### 8g. The fix — `tools/swdl-whitelist-patch.py`

Replaces the **last** live entry in place. **4 bytes change**, nothing else:

```
offset 0x38e43c: 86 12 -> CC 2E     (VID 0x1286 -> 0x2ECC)
offset 0x38e440: 89 81 -> 02 30     (PID 0x8189 -> 0x3002)
```

`bcdDevice` stays `0xFFFFFFFF`, the terminator and every byte after the table
are untouched, and `.text` is byte-identical to the original — the patch is
data-only. Cost: `0x1286:0x8189` stops being recognised, and we do not own
one.

Rejected alternative: appending after the terminator. The 12 bytes past it hold
`0x0813c740` / `0x0813c720`, code addresses that are almost certainly a
vtable pointer pair; zeroing them risks a crash on any virtual call.

Output `flashing_tools/p298-swd/swdl_linux_p298`. Verified: ELF headers and all
section headers identical, `.text` sha256 unchanged.

### 8h. The patch WORKS — one gate remains, 2026-10-14

Live run with the patched binary against the real board:

```
Begin to detect wtptp device....
Please plug your USB device
add DevName:/dev/bus/usb/001/023
LogFileName  is:.../wtp_dev_bus_usb_001_023.log
Get type of device failed
Open device failed
```

`wtp_dev_bus_usb_001_023.log` — **the per-device log that has never existed
before this patch.** The whitelist was indeed the gate. The device is now
found, opened, and spoken to.

The log is a complete handshake transcript:

```
openUsbPort  m_fdUsb = /dev/bus/usb/001/023....
 GetTargetMessage after sendpreamble
 GetWtpMessage...
 Platform ready status: 0
Getting Version...
 GetWtpMessage...
 Platform ready status: 0

Version: 3311
Date: 06012014
Processor: NZA3Unknown platform type

Get type of device failed
```

So: **USB opened, preamble accepted, platform-ready status 0, version
retrieved.** The board speaks the protocol and identifies itself. The only
remaining failure is platform recognition.

### 8i. The last gate — device platform enum

`CWtptpDownLoad::GetDeviceBootType` (0x081361f0):

```asm
81369fd: cmp  DWORD PTR [esi+0x1c4c],0x12     ; platform enum
8136a04: jbe  8136a30                          ; <=18 -> dispatch
8136a06: fprintf(...,"Unknown platform type")
...
8136a30: mov  eax,DWORD PTR [esi+0x1c4c]
8136a36: jmp  DWORD PTR [eax*4+0x83371dc]      ; 19-entry jump table
```

Enums 0, 6 and 12 also land on the `Unknown platform type` handler, so those
slots are invalid too.

> ### SOLVED 2026-10-15 — see `swdl-platform-enum.md`
>
> Everything below the jump table in this section is **superseded and was
> wrong**. Two errors in particular:
>
> * The claim that the enum "is copied out of a runtime-built record" that
>   cannot be read statically is **false**. The identity lookup keys are static
>   `std::string`s built from `.rodata` literals by the global constructor at
>   `0x0812FC40`. The full enum → identity table is recovered there.
> * The version/name pairings above are transposed. `NZA3` is
>   **version `3311`, date `06012014`** — not `3307`. There is no mismatch on
>   our board at all.
>
> The real cause is much simpler. The enum is **not** derived from the board;
> it comes straight from the BLF:
>
> ```
> [TIM_Configuration] Processor_Type = PXA1826
>   -> CBlfParser::GetPlatFormType     -> ePlatForm = 15
>   -> CIFL_SWD::Initilizeparameter     -> _InstanceParams+0x24
>   -> CWtptpDownLoad::InitializeBL    -> this+0x1C4C
>   -> GetDeviceBootType jump table[15]
> ```
>
> `PXA1802` → ePlatForm **6**, which is one of the dead stubs. `PXA1826` →
> ePlatForm **15**, whose handler tests exactly `06012014` / `NZA3` / `3311` —
> the identity our board reports.
>
> **The fix is a one-token BLF edit, not a binary patch:**
>
> ```diff
> -Processor_Type = PXA1802
> +Processor_Type = PXA1826
> ```
>
> `PXA1802` prints `NEZHA platform type`; `PXA1826` prints
> `NEZHA3 platform type`. The board's own processor string is `NZA3`, so the
> BLF was mis-declaring the SoC family all along.
>
> If the binary patch is still wanted as a fallback, the correct constant is
> `--enum 0x0F`; `0x12` selects the HLN4 / `3313` branch, which is wrong.

### 8j. Recovered protocol constants

Read from disassembly, useful if a native client is ever written:

| command | transmitted bytes | source |
|---|---|---|
| Preamble | `00 D3 02 2B` (4 B) | `CWtpComm::SendPreamble` 0x0816d820 |
| DataHeader | `2A <seq> 00 <04 if fast> 04 00 00 00 <size BE32>` (12 B) | `CWtpComm::DataHeader` 0x0816ea00 |
| VerifyImage | `27 00 00 00 01 00 00 00 <mode>` (9 B) | `CWtpComm::VerifyImage` 0x0816eca0 |
| UploadDataHeader | `32 00 00 00 10 00 00 00 <4 × u32>` (24 B) | `CWtpComm::UploadDataHeader` 0x0816f530 |

**Correction 2026-10-15.** This table previously listed the constants
byte-reversed (`D3 02 2B 02`, `04 00 00 2A`, `01 00 00 00 27`,
`10 00 00 00 32`). Those were the little-endian encodings of the compiler's
*dword stores*, not the bytes on the wire. The opcode is at **offset 0**.
Full opcode table in `swdl_linux-internals.md` §5.2.

Full symbol map for `CWtpComm` and `CWtptpDownLoad` is recoverable with
`readelf -sW swdl_linux | c++filt`; the binary is unstripped and carries DWARF.

An earlier revision of this file concluded the tools "cannot open this device"
because no `0x2ECC` constant exists. **That inference was wrong.**

Scanning every download-capable binary across all four toolchains for
interface GUIDs, in both ASCII and UTF-16:

| binary | GUIDs found |
|---|---|
| `SWDownloader.exe` | `f18a0e88-c30c-11d0-8815-00a0c906bed8` |
| `WtptpDownLoad.dll` (4.8.8.2 and SwUpgrade_C) | same |
| `SWDConsole.exe`, `FBFDownloader.exe` | same |
| `ONTIM_Marvell_MultiDL.exe` | three unrelated Office/VSTO GUIDs |

`f18a0e88-c30c-11d0-8815-00a0c906bed8` is **`GUID_DEVINTERFACE_USB_DEVICE`** —
the interface matching *every* USB device. No vendor-specific GUID appears
anywhere. No `VID_`/`PID_` hardware-ID string appears in any binary.

The tools therefore enumerate **all** USB devices, open each, and probe for
the WTPTP protocol. Nothing matches on `0x2ECC` or on the `0x1286` family.
The absence of a VID constant is by design and says nothing about
compatibility.

**Consequences:**

- The Windows products have no reason to reject `2ecc:3002` on version
  grounds. `SWDownloader.exe` 4.8.8.2 may work, and
  `Marvell_SwUpgrade_P`, `Marvell_SwUpgrade_C` and `ONTIM_Marvell_MultiDL` are
  all worth trying.
- The Linux failure has another cause. Leading candidate: a kernel driver
  claiming the device. If `cdc_acm`, `usb-storage` or similar binds to
  `2ecc:3002` at enumeration, the vendor tool can no longer reach it —
  documented as a real failure mode for Yuge ASR module upgrades ("Linux
  driver conflict"). `tools/usb-watch.py` now reports bound drivers, with a
  150 ms settle delay after enumeration.

The protocol probe itself is the real version gate, and it has not been
observed succeeding on either OS. That is the honest state.

---

## 7. Offline validation tooling

The vendor toolchain ships Linux binaries that run the same parser without a
router, USB or root:

| binary | command |
|---|---|
| `Utilities/TimBuilder/TimBuilder` | `-r BLF_filename [-f tim_folder]` |
| `Utilities/MakeDownloadPackage/MakeDownloadPackage` | `-r BLF_filename -o package.zip` |
| `Utilities/FBFMake/FBFMake` | `-r BLF_filename -f fbf_folder` |
| `Utilities/RomCopy/RomCopy_linux` | `-b BLF_filename -o out.bin` (NAND not supported) |

`MakeDownloadPackage` reproduces `swdl_linux`'s full DKB failure chain and is
therefore the faithful oracle for this bug.

`LC_ALL=C LANG=C` is mandatory; without it they abort with
`locale::facet::_S_create_c_locale name not valid` before opening the file.
`swdl_linux` additionally needs root (`rmmod wtptp` plus USB); run without it
it exits silently with no banner.
