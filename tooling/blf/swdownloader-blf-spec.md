# `.blf` configuration format — Marvell / ASR SWDownloader family

Reference specification recovered by reverse engineering
`Marvell_SWDownloader_v4.8.8.2/swdl_linux` (ELF32 i386, statically linked,
not stripped, ships DWARF `debug_info`) and confirmed by bisection against the
tool's own parser.

Everything below is a **format description**. The analysis of *why* this
format exists and what blocked us is summarised in the final section and lives
in full in the main workspace repository.

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

## 6. Where the write path stops

The `.blf` in this directory parses cleanly and passes offline validation, but
`swdl_linux -D` then fails:

```
Platform is NEZHA.....
Prepare FBF list..... / Prepare FBF list return.....
Preparing NEZHA DKBlist
IFLSWD exception: OBMImage is NULL in prepareDKB list
```

Thrown by `CIFL_SWD::PrepareTwoStagDKBlist()`:

```c
CTimDescriptor::ImagesList();     /* of  this + 0x3804  -- the DKB TimDescriptor */
if (list.empty())
    throw "OBMImage is NULL in prepareDKB list";
```

That list is **not** the `.blf` image list. It is a boot-ROM *download package*
TimDescriptor populated by `CIFL_SWD::GenerateDKBTim()`, and it expects the
four `WTMI`/`TZII`/`TIMH`/`OBMI` entries listed in §3.

Verified with `gdb` that the parser stores our identifiers correctly
(`ImageId` setter called with `OBMI`, `TIMH`, `TIMH`; member at `this + 0x44`).
Adding synthetic `WTMI` / `TZII` entries, and toggling `BLF_Version` across all
four accepted values, `Tim_Included` and `Enable`, all leave the error
unchanged.

`Marvell_SwUpgrade_C_v1.03`, `Marvell_SwUpgrade_P_V1.05` and
`ONTIM_Marvell_MultiDL_v1.4.3.72` ship the same `IFL_SWD` library and contain
the same four identifiers and the same error string, so none of them provides
an alternative route.

**Conclusion:** SWDownloader does not flash "a list of MTD dumps". It assembles
a boot-ROM download package from Marvell-internal support blobs. `TIMH` and
`OBMI` are available from this repository's dumps; `WTMI` and `TZII` are not
part of any MTD partition and are not obtainable.

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
