# `swdl_linux` 4.8.8.2 — how the Linux download engine actually works

Reverse-engineered from the shipped artifact only:
`Marvell_SWDownloader_v4.8.8.2/swdl_linux`

| | |
|---|---|
| format | ELF 32-bit LSB, Intel i386, **statically linked** |
| size | 4 622 608 bytes |
| sha256 | `cd01624ac5af1d26a3e728a5d2eec74ab6d29b035d990987c3d3dc7c1f96aa80` |
| buildid | `1e7af8ad0c0716c91138b4c45de27f230f20beb6` |
| self-reported version/date | `4.8.8.2` / `2015-8-3` |
| toolchain | GCC 4.4.3 (C++), glibc/eglibc 2.11.1, static |
| symbols | **not stripped**, 6 743 `FUNC` symbols, full DWARF 2 `debug_info` |
| build path (from DWARF) | `/home/zqq/share/SWD_Linux_unicode/src/Linux/SWDownloader/SWDTool.cpp` |

Because it is unstripped with debug info, every statement below is a direct
read of the code, not a guess. Analysis method: `readelf -sW | c++filt`,
`objdump -d`, DWARF CU/line tables, and manual decoding of `.rodata` and the
`.data` tables. Ghidra's full auto-analysis exceeds its startup deadline on
this 4.6 MB static binary, so symbol-driven static analysis was used instead —
which is strictly more reliable here because the symbol names are the original
Marvell names.

Related: `swdownloader-blf-spec.md` (the `.blf` file format), §8 for the
whitelist and platform-enum work.

---

## 1. Executive summary

`swdl_linux` is Marvell's complete WTPTP (Wireless Transfer Protocol) download
engine. The Windows `SWDownloader.exe` is this engine plus an MFC/Codejock GUI.
There is no Linux GUI in the vendor package.

The run is a straight-line pipeline:

```
main
 ├─ already_running()                      flock on /var/run/swdlinux.pid
 ├─ system("rmmod wtptp.ko")               ← why root is needed even before USB
 ├─ ParseCommandLine()                     -D -U -S -T -N
 ├─ GetAppPath()                           readlink("/proc/self/exe")
 ├─ ReadBLF()                              CBlfParser::ParseBlfFile
 ├─ InitializeBL(_InstanceParams*)         builds temp/ + AbsBlfTemp/, TIM, FBF, DTIM
 │    └─ pthread_create(WtptpDeviceDetectThread)
 └─ wait on condvar  ── thread: hotplug netlink → FindAllSupportedDevices
                                   → OpenUsbPort → SendPreamble → GetTargetMessage
                                   → GetVersion → GetDeviceBootType → WTPTPDownLoad
                                                             → DownloadImage
```

---

## 2. Command line

`ParseCommandLine(int argc, char** argv)` @ `0x08048370`, 276 bytes. It walks
`argv[1..]`, requires each to start with `'-'`, takes the option letter as
`argv[i][1]` (the rest of the token is ignored) and dispatches through a jump
table at `0x0832CC10` indexed by `letter - 'D'`.

| flag | effect | storage |
|---|---|---|
| `-D <file>` | download mode; `<file>` is the BLF | `gBlfFileName` `0x083F97E0` |
| `-U <file>` | upload/read-back mode; `<file>` is the upload `.cfg` | `gUploadCfgName` `0x083F97E4` |
| `-S` | single download | `gSingleDownload` `0x083F97EE` |
| `-T <secs>` | timeout; `atoi(argv[i+1])` | `gTimeout` `0x083F97F4` |
| `-N` or `-n` | do **not** auto-save the temp BLF | `gNoAutoSaveBlf` `0x083F97F0` |
| anything else | `Error: Command line option -%c is invalid!` and `argc<=2` usage dump | — |

Note `-n` is accepted as a synonym for `-N`: the jump table maps both letters to
the same handler.

Usage text is exactly:

```
./swdl_linux -D BLF_filename  [-T Timeout(seconds)] [-N]
./swdl_linux -D BLF_filename -S  [-T Timeout(seconds)] [-N]
./swdl_linux -U Upload_configura_filename
```

After argument parsing, `main` prints the banner
(`SWDL Version: %s` / `SWDL Date: %s`, values read from the pointer globals
`0x083B6828` → `"4.8.8.2"` and `0x083B6824` → `"2015-8-3"`).

`-D` and `-U` together are rejected:
`Error command format:user can't do upload and download at the same time`.

---

## 3. Singleton, privilege and path resolution

### 3.1 Singleton

`already_running()` @ `0x0804828C`:

```c
fd = open("/var/run/swdlinux.pid", O_RDWR|O_CREAT /*0x42*/, 0644 /*0x1A4*/);
if (fd < 0) { syslog(3, "can't open %s: %s", path, strerror(errno)); exit(1); }
if (lockfile(fd)) {                 /* fcntl(fd, F_SETLK=6, {F_WRLCK,0,0,0}) */
    if (errno == EACCES || errno == EAGAIN) { close(fd); return 1; }  /* busy */
    syslog(3, "can't lock %s: %s", path, strerror(errno)); exit(1);
}
return 0;
```

The lock is a POSIX whole-file write lock. A `SIGKILL`ed run releases it
automatically, so the "stale pid file blocks the next run" failure mode
described in `linux_gui/GUI_README.md` does **not** apply to a killed process —
it only applies if another live process holds it. (The pid file itself is
never deleted on exit.)

### 3.2 Why root is required

Not `getuid()`. `main` calls `system("rmmod wtptp.ko")` before doing anything
else, and the engine later opens `/dev/bus/usb/*` directly via usbdevfs
`ioctl`. Either alone forces root.

### 3.3 Write-permission probe

`HaveWritePermission(const char* path, bool isFile)` @ `0x08082140`:

* `isFile == false`: `fopen(path, "w+")` → `fclose` → `DelFile(path)`.
* `isFile == true`: builds `path + "/marvellswd_testfile"` (19 chars + NUL),
  `access(..., F_OK)`; if absent, `fopen(...,"w+")`, `fclose`, `DelFile`.

So the probe file is created **inside the directory being tested**, not at the
filesystem root. `linux_gui/GUI_README.md` §4.1 states the opposite; that claim
is wrong.

### 3.4 Path resolution — `/proc/self/exe`, not CWD

`GetAppPath()` @ `0x08080940`:

```c
readlink("/proc/self/exe", buf, 0x400);
*strrchr(buf, '/') = '\0';
return std::string(buf);
```

Every internal path is built from this:

```
<appdir>/temp/…
<appdir>/AbsBlfTemp
<appdir>/TempSplit
<appdir>/fbf.bin
<appdir>/swdl_linux_wtptp_device_detect.log
```

`CWtptpDownLoad::InitializeBL` @ `0x0813AA90` independently does the same
`readlink("/proc/self/exe")`.

**`linux_gui/GUI_README.md` §4.3 ("`./AbsBlfTemp` and `./temp` are resolved
relative to CWD") is incorrect.** The working directory is irrelevant; what
matters is where the `swdl_linux` binary lives. This is worth knowing because
the engine can be launched from anywhere as long as the package layout is intact
next to the binary.

### 3.5 BLF image paths

`SetAbsolutePath(char* p, bool)` @ `0x08082C30`:

```c
if (!IsPathRelative(p)) return;          /* already absolute */
strcpy(buf, GetAppPath().c_str());
strcat(buf, p);
```

`IsPathRelative()` @ `0x080807A0` strips spaces into a scratch copy and returns
`buf[0] != '/'`. Two consequences:

1. A **relative** image `Path` in the BLF is resolved against the **binary's
   directory**, not the CWD.
2. `IsPathRelative` only understands POSIX absolute paths. A Windows path such
   as `Z:\Notion_P298_ASR1802S\...\mtd0.bin` is classified *relative* and gets
   the app path glued in front of it, producing garbage. That is exactly what
   the shipped `AbsBlfTemp/Abs_p298_fixed.blf` still contains — those `Z:\…`
   paths can never resolve on Linux.
3. `str_remove_spaces()` is applied to the *classification copy*, not the
   original, so spaces in the real filename are preserved. (Worth remembering
   if a path is ever rejected.)

---

## 4. USB transport

Class `CBulkDevNonDrv_Linux` @ `0x0813B170`… (16 methods). This is Marvell's
own port of the Windows `WTPTP.sys` bulk driver; it does **not** use libusb.

### 4.1 Enumeration — `FindAllSupportedDevices` @ `0x0813B9A0`

```
opendir("/dev/bus/usb")
  for each entry whose name starts with a digit (bus number):
    opendir("/dev/bus/usb/<bus>")
      for each entry whose name starts with a digit (device number):
        fd = open("/dev/bus/usb/<bus>/<dev>", O_RDONLY)
        n  = read(fd, buf, 0x400)
        walk USB descriptors in buf:
          type 0x01 DEVICE      → idVendor  = buf[8], idProduct = buf[10]
                                  ChecHWIds(vid, pid, 0xFFFFFFFF)
          type 0x02 CONFIG      → bNumInterfaces = buf[4]
          type 0x04 INTERFACE   → bNumEndpoints  = buf[4]
          type 0x05 ENDPOINT    → record bulk IN / bulk OUT endpoint addresses
```

Reading the usbfs device node is how it obtains the raw descriptors — there is
no `USBDEVFS_GET_DESCRIPTOR` ioctl involved.

### 4.2 The whitelist — `ChecHWIds(long vid, long pid, long bcd)` @ `0x0813B220`

```asm
mov  edx,[ecx]            ; entry[0].vid  -- read ONLY as a "table non-empty" test
test edx,edx
je   fail
lea  eax,[ecx+0xc]        ; scan starts at entry[1]
loop:
  mov  edx,[eax]
  mov  ecx,eax
  add  eax,0xc
  test edx,edx            ; vid == 0 terminates
  je   fail
  cmp  edx,ebx            ; vs wanted vid
  jne  loop
  cmp  DWORD PTR [ecx+4],esi   ; vs wanted pid
  jne  loop
  cmp  edi,0xffffffff
  je   hit
  cmp  DWORD PTR [ecx+8],edi   ; vs wanted bcdDevice
  je   hit
hit:
  mov  eax,1
```

Table: global pointer at `0x08337304` → array at **va `0x083D72E0`**
(file offset `0x38E2E0`), stride 12 bytes `{vid, pid, bcdDevice}`, terminated
by `vid == 0`.

**Correction to `swdownloader-blf-spec.md` §8f:** the prose there says "30 live
entries". The table has **30 stored entries but only 29 are reachable** — the
loop starts at `entry+0xC`, so `entry[0]` is consumed only as the
non-empty-table flag and is never compared. `0x8086:0xE001` sits at `entry[0]`
and can never match. The reachable set is:

| VID | PIDs | bcdDevice |
|---|---|---|
| `0x8086` Intel | `0xC001`, `0xD001` | `0xFFFFFFFF` (wildcard) |
| `0x1286` Marvell | `0x8100`–`0x8106`, `0x8111`–`0x8118`, `0x8128`, `0x8129`, `0x812A`, `0x8130`, `0x8164`, `0x8170`, `0x8165`, `0x8174`, `0x8175`, `0x8181`, `0x8182`, `0x8189` | `0xFFFFFFFF` |

`0x2ECC` is absent, so the conclusion in §8f stands. Nothing changes for
`tools/swdl-whitelist-patch.py`, which correctly counts 29 reachable and
replaces the last reachable entry (`0x1286:0x8189`) in place.

### 4.3 Open / endpoints

`OpenDev(_USB_HANDLE*)` @ `0x0813B660`:

```c
fd = open(devpath, O_RDWR /*0x2*/);
n  = read(fd, buf, 0x400);
GetPipeOfDev(buf, n, &bulkIn /*+0x204*/, &bulkOut /*+0x208*/, &third /*+0x200 slot*/);
handle->fd          = fd;   /* +0x200 */
handle->bulkInEp    = in;   /* +0x204 */
handle->bulkOutEp   = out;  /* +0x208 */
```

`GetPipeOfDev(char* desc, int len, int*, int*, int*)` @ `0x0813B280` re-walks
the descriptor tree itself: DEVICE → verify VID/PID via `ChecHWIds` →
CONFIG → every INTERFACE → every ENDPOINT (type 0x05), assigning the endpoint
number for direction IN to one out-param and OUT to the other.

`_USB_HANDLE` layout as used:

| offset | meaning |
|---|---|
| `+0x200` | file descriptor (`-1` when closed) |
| `+0x204` | bulk **IN** endpoint number |
| `+0x208` | bulk **OUT** endpoint number |

### 4.4 Transfers — one ioctl, two retry loops

Both `Write` @ `0x0813B520` and `Read` @ `0x0813B470` use exactly one ioctl:

```
request = 0xC0105502            /* _IOR('U', 0x02, struct usbdevfs_bulktransfer) */
struct usbdevfs_bulktransfer {
    unsigned int ep;     /* +0x00  endpoint          */
    unsigned int len;    /* +0x04  length            */
    unsigned int timeout;/* +0x08  ms (0 = infinite)*/
    void         *data;  /* +0x0C  buffer            */
};
```

Legacy **usbdevfs**, not `/dev/bus/usb` `ioctl(USBDEVFS_BULK)`.

* `Read`: loops until the requested byte count is satisfied. A failing ioctl is
  retried **6 times** with `usleep(10000)` between attempts, then returns `-1`.
* `Write`: **no retry** — one ioctl; if the return value differs from the
  requested length it logs `ERROR: n = %d, errno = %d (%s)` and returns `-1`.

`CBulkDevNonDrv_Linux::WaitDev()` @ `0x0813B7F0` and
`CWtptpDownLoad::init_hotplug_sock()` @ `0x0812FAE0` use a **netlink
`NETLINK_KOBJECT_UEVENT`** socket for add/remove notifications — that is where
the glibc `__netlink_*` symbols come from.

---

## 5. The WTP protocol (`CWtpComm`, 36 methods)

### 5.1 Framing

Every command is a flat little-endian byte frame built on the stack:

```
off 0   u8   opcode
off 1   u8   sequence (only Data / DataHeader; ++this->m_nSeq, 8-bit, wraps)
off 2   u8   flags
off 3   u8   fast-download / sub-code
off 4   u32  parameter or inline data pointer
off 8   ...  payload
```

Responses are read into a `WTPSTATUS` whose `Status` is at **offset 3**.
`resp[3] == 1` means **NACK**; every command method tests exactly that.

### 5.2 Opcode table (recovered)

| opcode | method | request | response | payload |
|---|---|---|---|---|
| `0x00` | `SendPreamble` | `00 D3 02 2B` (4 B) | 0x26 = 38 B | — |
| `0x20` | `GetVersion` | 8 B (`20 00 00…`) | 0x1E = 30 B | target version / date / processor string |
| `0x22` | `Data` | `8 + N` B | 6 B | `DataHeader`+payload are copied into one contiguous buffer, so the 8-byte header at `cmd[0..7]` and the payload at `cmd[8..]` are a single bulk transfer |
| `0x26` | `SelectImage` | 8 B | 0x0A = 10 B | **a query**: returns `resp[6..9]` as a big-endian u32 via the out-pointer |
| `0x27` | `VerifyImage` | 9 B | 6 B | `27 00 … 01000000 <mode>` — `mode` at offset 8 |
| `0x2A` | `DataHeader` | 12 B | 0x0A = 10 B | `2A <seq> 00 <4 if fast> 04000000 <size BE32>` — size is 4 bytes **big-endian** at offset 8 |
| `0x2B` | `GetWtpMessage` | 8 B | — | the target's async notification channel |
| `0x30` | `Done` | 8 B | — | |
| `0x31` | `Disconnect` | 8 B | — | |
| `0x32` | `UploadDataHeader` | 24 B (`0x18`) | 6 B | `32 00 … 10000000` + four u32 from `_UPLOAD_DATA_PARAMS` |
| `0x33` | `UploadData` | 8 B + N | — | |
| `0x34` | `GetProtocolVersion` | 8 B | 0x0A = 10 B | `resp[3]=status`, `resp[5]=len`, `resp[6..9]` = major, minor, u16 build |
| `0x35` | `GetParameters` | 8 B | 0x16 = 22 B | `resp[5]` must be 0x10; then four u32: **Buffer Size**, Rsvd1, Rsvd2, Rsvd3 |

> **Correction to `swdownloader-blf-spec.md` §8j.** That table lists the
> constants byte-reversed (`D3 02 2B 02`, `04 00 00 2A`, `01 00 00 00 27`,
> `10 00 00 00 32`). Those are the little-endian encodings of the compiler's
> *dword stores*, not the transmitted bytes. The opcode is at **offset 0**.

### 5.3 `SendPreamble` @ `0x0816D820`

```c
u8  cmd[0x4008] = {0};
cmd[1] = 0xD3; cmd[2] = 0x02; cmd[3] = 0x2B;
memcpy(cmd + 4, "\x2B\x02\xD3\x00", 4);   /* compiler dword store; outside the 4-B length */
if (!WriteUSB(cmd, 4)) return false;     /* sends 00 D3 02 2B */
ReadUSB(status, 0x26);                    /* 38 bytes */
if (status[0] != 0 && status[1] != 0xD3 && status[2] != 0x02)
    warn("SendPreamble returned a wrong status");
if (status[3] == 1) { "Error: SendPreamble returned a NACK!"; return false; }
"ReadUSB is ok in sending preamble...";  return true;
```

This is the one command whose first byte is not the opcode — it is `0x00` and
the preamble magic `D3 02 2B` follows. The response mirrors it:
`00 D3 02 <status>`.

### 5.4 Retry / timeout policy

| layer | retries | delay |
|---|---|---|
| `WriteUSB` | 20 (`cmp ebx,0x14`) | `usleep(1000)`; on give-up prints `Error: Write failed with error %s.` |
| `ReadUSB` | 10 (`cmp edi,0xa`) | `usleep(1000)`; on give-up prints `Error: ReadUSB failed with error %s.errno = %d` |
| `CBulkDevNonDrv_Linux::Read` | 6 (`cmp [ebp-0x30],0x6`) | `usleep(10000)` |
| `CBulkDevNonDrv_Linux::Write` | 1 | — |

`WriteUSB`/`ReadUSB` also abort immediately with `"Thread is to be exit"` when
`this->m_bExitThread` (`+0x424`) is set.

Default USB packet size is set in the `CWtpComm` constructor:
`this[+0x630] = 0xFF8` (**4088**). `SetPacketSize(u16)` @ `0x0816D010` does
`((n + 0x3F) & 0x1FFC0) - 8`, i.e. round up to 64 and subtract the 8-byte
`Data` header. At the default that leaves **4080 payload bytes** per transfer.

---

## 6. Object model (78 non-`std` classes)

Grouped by responsibility:

| group | classes |
|---|---|
| protocol | `CWtpComm`, `CWtptpDownLoad`, `CConvertLT2B` |
| USB | `CBulkDevNonDrv_Linux`, `CUsb`, `CUsbVendorReq`, `CUart` |
| BLF parsing | `CBlfParser`, `CIFL_SWD`, `CConsumerID` |
| TIM model | `CTimDescriptor`, `CTimLib`, `CImageDescription`, `CImageBuilder`, `CImageMapInfo`, `CImageMaps`, `CFlashImage`, `CFlashDevice`, `CFlasherBinary`, `CFBF_File`, `CFBF_File::INTEL_FLASHER_INPUT_FILE` |
| TIM reserved packages | `CXfer`, `CTBRXferSet`, `CInstructions`, `CGpio`, `CGpioSet`, `CGPPInitialization`, `CGPPOperations`, `CDDRInitialization`, `CDDROperations`, `CVendorDDRInitialization`, `CTzInitialization`, `CTzOperations`, `CResume`, `CResumeDdr`, `CEscapeSeq`, `CEscapeSeqV2`, `CAutoBind`, `CBbmt`, `CCoreReset`, `CCustTransferAPInfo(Set)` — all deriving `CErdBase` with `PackageName`/`PackageEndName`/`PackageSize`/`AddPkgStrings`/`Migrate` |
| crypto | `CRSAKey`, `CECCKey`, `CKey`, `CEncryptKey`, `CDigitalSignature` |
| platform register maps | `AspenStrings`, `MMP2Strings`, `MMP3Strings`, `TTC1Strings`, `SdramStrings` |

### 6.1 TIM reserved-package names

Recovered from each `AddPkgStrings` (`PackageName` = 1st literal,
`PackageEndName` = the object's own name):

| class | literals |
|---|---|
| `CXfer` | `DATA_ID`, `LOCATION` |
| `CTBRXferSet` | `XFER_LOC`, `NUM_DATA_PAIRS` |
| `CInstructions` | `WRITE` |
| `CGpio` | `ADDRESS`, `VALUE` |
| `CGpioSet` | `NUM_GPIOS` |
| `CGPPInitialization` / `CGPPOperations` | `NumOps`, `NumInst` / `Value` |
| `CDDRInitialization` / `CDDROperations` | `NumOps`, `NumInst` / `Value` |
| `CVendorDDRInitialization` | `NumOps`, `NumInst` |
| `CTzInitialization` / `CTzOperations` | `NumOps`, `NumInst` / `Value` |
| `CResume` / `CResumeDdr` | `IMAGE_ADDRESS` / `RESUME_DDR_ADDR`, `RESUME_DDR_PARAM`, `RESUME_DDR_FLAG` |
| `CEscapeSeq` / `CEscapeSeqV2` | `ESCAPE_SEQ_TIMEOUT_MS` / `ESCAPE_SEQV2_GPIONUMBER`, `ESCAPE_SEQV2_GPIO_TRIGGERVAL`, `ESCAPE_SEQV2_MFPR_NUM`, `ESCAPE_SEQV2_MFPR_ADDR`, `ESCAPE_SEQV2_MFPR_VALUE` |
| `CUsb` / `CUart` | `PORT`, `ENABLED` |
| `CUsbVendorReq` | `wVALUE, bREQUEST, bREQUEST_TYPE`, `wLENGTH, wINDEX`, `data` |
| `CCustTransferAPInfo(Set)` | `ADDRESS`, `VALUE` / `NUM_TRANSFERTAPINFOCFGS` |
| `CImageMapInfo` | `Image_ID`, `Type_ID`, `Flash_Address_Lo`, `Flash_Address_Hi`, `Partition` |
| `CImageMaps` | `NUM_MAPS` |
| `CAutoBind` / `CBbmt` / `CConsumerID` / `CCoreReset` | `AUTOBIND` / `BBMT Type` / `num PIDs` / `Core ID`, `Address Mapping` |

### 6.2 Recognised `Processor_Type` strings

From `.rodata` @ `0x0832D363`…`0x0832D4B2`:
`PXA920_Old`, `PXA92x`, `MMP2`, `MMP3`, `PXA168`, `PXA30x`, `PXA31x`, `PXA32x`,
`PXA688`, `PXA91x`, `PXA93x`, `PXA94x`, `PXA95x`, `PXA2128`, `PXA988`,
`PXA1088`, `PXA1202`, `PXA1802`, `PXA1928`, `PXA1920`, `PXA1822`, `PXA1826`,
`PXA1908`, `PXA1936`, `PXA1956`, plus `PXA1U88`, `PXA1928_A0`, `PXA92X`.
Our board uses `PXA1802` (the *Aspen* family — `AspenStrings`).

Flash families (`GetFlashType`): `SPI-NOR`, `SPI-NAND`, `ONENAND`, `x16_NAND_BCH`,
`x8_NAND_BCH`, `NAND_FLASH_X16_BCH_OPT1`, `NAND_FLASH_X8_BCH_OPT1`,
`NAND_FLASH_X16_NO_ECC`, `NAND_FLASH_X8_NO_ECC`, `x16_OneNAND`,
`x16_NAND_Hamming`, `NAND_x8_Hamming`, `BOOTSPI_FLASH`, `SPI_Nand_Flash`.

---

## 7. Image preparation pipeline

`CIFL_SWD` (97 methods) + `CWtptpDownLoad::prepareDownloadImg`
@ `0x0813A100` build everything before any USB traffic:

```
CIFL_SWD::ProcessBLF        → ParseBlfFile
CIFL_SWD::Initilizeparameter
CIFL_SWD::CheckDdrFlashMapConfig / CheckFlashImagelistcfg
CIFL_SWD::CheckImageOverlapwithMultipleDDRandEMMC
CIFL_SWD::PrepareImgData    → CompressSpecLzmaImage, SplitSparseImage, SplitSingleImage
CIFL_SWD::GenerateTim       → PrepareTimImgList, PrepareDKBTimImgList,
                               PrepareFBFTimImgList, ProcessTimDsPublicKey,
                               ProcessDtimENCK, GenerateDTIM
CIFL_SWD::GenerateTIMImgBin
CIFL_SWD::CreateFBFBin*     → CreateNTIMForFBF, CreateFbfTimHeader,
                               GenerateMuitipleFBFFiles
CIFL_SWD::PreTempForDownload
CWtptpDownLoad::ParseTim    → list<stImageFileByDDR*>
```

Files it writes next to the binary:

```
<appdir>/temp/TimHeader.bin     <appdir>/temp/DKB.bin
<appdir>/temp/Dkb.bin           <appdir>/temp/TZII.bin
<appdir>/temp/WTM_h.bin         <appdir>/temp/OBM_h.bin
<appdir>/temp/TZII_h.bin        <appdir>/temp/cfg.ini
<appdir>/temp/FBF_h.bin         <appdir>/temp/FBF<n>_h.bin
<appdir>/temp/<ddr>_FBF_h.bin   <appdir>/temp/<ddr>_FBF<n>_h.bin
<appdir>/temp/<ddr>_FBF<c>_h.bin
<appdir>/AbsBlfTemp/Abs_<name>.blf
<appdir>/fbf.bin                <appdir>/TempSplit
<appdir>/swdl_linux_wtptp_device_detect.log
<appdir>/wtp_dev_bus_usb_<bus>_<dev>.log       per device
<appdir>/<…>_failed_wtp.log                   on failure
```

It also shells out (`system()`) for `mkdir`, `mkdir -p`, `rm -rf`, `rm -f`,
`mv`, `zip -r -j`, `unzip -o -j -d` — so `/bin/sh`, `zip` and `unzip` must
exist.

### 7.1 FBF (`MasterBlockHeader`) layout

From the parser's own trace strings in `.rodata`:

```
MasterBlockHeader.Format_Version       (u32)
MasterBlockHeader.Size_of_Block        (u32)
MasterBlockHeader.Bytes_To_Program     (u32)
MasterBlockHeader.Bytes_To_Verify      (u32)
MasterBlockHeader.Number_of_Bytes_To_Erase (u32)
MasterBlockHeader.Main_Commands        (u32)
MasterBlockHeader.nOfDevices           (u32)
MasterBlockHeader.DLerVersion          (u32)
  DeviceHeader[nOfDevices].DeviceFlags      (u32)
  DeviceHeader[nOfDevices].nOfImages        (u32)
    image_struct[].Flash_partition          (u32)
    image_struct[].commands                 (u32)
    image_struct[].First_Sector             (u32)
    image_struct[].length                   (u32)
    image_struct[].Flash_Start_Address      (u64)
    image_struct[].ChecksumFormatVersion2   (u32)
```

### 7.2 Logging

Progress is emitted with a `\r`-style spinner, which is what the existing GUI
scrapes:

```
|%s:Download percentage is:%d  |
|%s:Burning Flash percentage is:%d  |
|%s:Uploading percentage is:%d      |
|%s:Erasing all flash percentage is:%d      |
|%s:Burn Successfully                |
|%s:Upload Successfully                |
|%s:Abort in download          |
```

Two sinks: `stdout`/`stderr` and `this->m_fpLog` (`FILE*` at `this+0x638`),
which is `<appdir>/swdl_linux_wtptp_device_detect.log` or the per-device
`wtp_dev_bus_usb_….log`. Every protocol method logs its first seven response
bytes:

```
"\nBytesRead : %d pWtpStatus:%x %x %x %x %x %x %x"
```

---

## 8. Device handshake and download sequence

### 8.1 `WtptpDeviceDetectThread` @ `0x08137460`

Runs for the whole session. Opens the netlink hotplug socket, and on each
`add`/`remove` event calls `FindAllSupportedDevices` + `OpenUsbPort`, creating
a per-device log `%s/wtp%s.log`. Adds the device to the display list
(`AddDisplayInfoDevice`), then hands off to `GetDeviceBootType`.

### 8.2 `GetDeviceBootType(_DEVICE*)` @ `0x081361F0`

```
IsLittleEndian()
SetMessageMode(...)
OpenUsbPort()
SetPortType(...)
SendPreamble()          → " Open Usb Port in GetDeviceBootType failed"
                       → "SendPreamble failed in GetDeviceBootType"
GetTargetMessage()      → "  GetTargetMessage after sendpreamble"
GetVersion()            → "  Getting Version..."
```

then prints

```
"\nVersion: %c%c%c%c\n"        e.g. 3311
"\nDate: %08X\n"               e.g. 06012014
"\nProcessor: %c%c%c%c\n"     e.g. NZA3
```

looks the (version, date) pair up in the runtime-built identity table, and
dispatches through a **19-entry jump table** at `0x083371DC` indexed by the
platform enum at `[esi+0x1C4C]`. `cmp DWORD PTR [esi+0x1c4c],0x12 / jbe`
before the jump — anything above 18 falls to `"Unknown platform type"`.
This is the gate described in `swdownloader-blf-spec.md` §8i.

On failure the thread sleeps and retries (`sleep(1)` observed between
`SetBootDeviceType` attempts).

### 8.3 `WTPTPDownLoad(_DEVICE*)` @ `0x081348A0`

```
GetFileSize() → SetPacketSize(buffer) → SetPortType(...)
loop:
  DownloadImage(dev, ...)          @ 0x08132F90
  GetWtpMessage()                  → drain target notifications
  ValidateWTPTPMSGReport()
  if error: write "<...>_failed_wtp.log", rename it, retry
  break on success
```

### 8.4 `DownloadImage(...)` @ `0x08132F90`

Exact command order recovered from the call list:

```
SendPreamble()
GetVersion()
SelectImage(&imageId)          → query the image the target wants
DataHeader(size, &status, fast)
    read back: bufferSize / FDSwitch / reserved
    if fast: "*******NOTE:: Fast Download is NOW activated!"
    else:    "Fast Download is not activated"
loop over the image in `packetSize` chunks:
    isFastDownload() / isLastData()
    Data(buf, len)             [WriteUSBWithNoRsp in fast mode]
    AckWithImageName(...)
VerifyImage(mode)
Done()
"Download data finished"
"Completed DownLoad file: %s"
```

`WriteUSBWithNoRsp` @ `0x0816D0C0` is the pipelined bulk-OUT variant used when
the target advertises fast download (gated by `this[+0x63C]`); it still retries
20× at 1 ms. In normal mode `Data` goes through `WriteUSB` and waits for the
6-byte reply each chunk.

---

## 9. Upload / read-back (`-U`)

`CUploadCfgParser::ParseUploadCfgFile` @ `0x081297D0` then
`CWtptpDownLoad::UploadDataFromTarget` @ `0x08137F00` +
`BootLoaderUploadThread` @ `0x08138FB0`:

```
UploadDataHeader(&params)   opcode 0x32, 24 B, reads 6 B back
   resp[3] must be 0
   resp[6..9] = error code, looked up in a 1024-entry x 0x68-byte
               name table at 0x083D78E0 → printf("Code: %s")
UploadData(buf, len)         opcode 0x33
```

The per-area parameters the parser reads are:

```
SubType / Partition / Offset / DataSize / ResetUE / ResetTimeDelay
```

and the run prints `Begin to upload data from 0x%x to 0x%x to file %s`,
`Uploading file percentage is:%d`, `Finish saving upload data file %s`.
Our repo's `flash_read_mtd0.cfg` / `flash_read_all.cfg` drive this path.

---

## 10. Corrections to existing project notes

| file / section | claim | reality |
|---|---|---|
| `swdownloader-blf-spec.md` §8f | "30 live entries" | 30 **stored**, 29 **reachable** — `entry[0]` (`0x8086:0xE001`) is skipped by the `+0xC` loop start and can never match |
| `swdownloader-blf-spec.md` §8j | Preamble `D3 02 2B 02`, DataHeader `04 00 00 2A`, … | opcode is at offset 0: Preamble `00 D3 02 2B`, DataHeader `2A …`, VerifyImage `27 …`, UploadDataHeader `32 …` |
| `linux_gui/GUI_README.md` §4.1 | probes root by creating `/marvellswd_testfile` at the filesystem root | `HaveWritePermission` appends `/marvellswd_testfile` to the **directory under test** |
| `linux_gui/GUI_README.md` §4.2 | a stale `/var/run/swdlinux.pid` makes the next start refuse to run | the lock is `fcntl(F_SETLK)`, released by process death; only a *live* holder blocks |
| `linux_gui/GUI_README.md` §4.3 | `./temp` and `./AbsBlfTemp` resolve against CWD | they resolve against `readlink("/proc/self/exe")` — the binary's directory |

---

## 11. Open questions

* ~~**Platform enum mapping.**~~ **RESOLVED 2026-10-15** — see
  `swdl-platform-enum.md`. The identity keys are static `std::string`s built by
  the global constructor; the enum itself comes from the BLF's
  `Processor_Type`, and `PXA1826` → 15 is the branch that matches this board.
  `tools/swdl-platform-patch.py` should use `--enum 0x0F`, not `0x12`.
* **Opcode `0x00`.** `SendPreamble` is the only command whose byte 0 is not an
  opcode. Whether that is a reserved opcode or a 1-byte offset in an earlier
  protocol revision is not determinable from this binary alone.
* **Protocol v3 framing.** `GetProtocolVersion` reports major/minor/build; the
  request frames look identical across versions in this code, so any
  version-dependent framing must be entirely target-side.
* **`_UPLOAD_DATA_PARAMS` field semantics** beyond the four u32 copied at
  `UploadDataHeader` offset 8.
* **Windows/MFC side.** `SWDownloader.exe` (21 MB, MSVC) was not analysed here;
  §1's claim that it is "this engine plus a GUI" rests on string/RTTI
  comparison and has not been verified at the code level.
