# SOLVED — the platform enum is the BLF's `Processor_Type`, and it was wrong

Date: 2026-10-15 · Tool: `readelf -sW | c++filt`, `objdump -d`, static analysis
of `Marvell_SWDownloader_v4.8.8.2/swdl_linux` (unstripped, DWARF 2).

This closes the "last gate" of `swdownloader-blf-spec.md` §8i. It was stated
there as *not resolvable statically*. That was wrong: the lookup keys are
static `std::string`s initialised from `.rodata` literals by the global
constructor. Only the **dispatch target** of the identity lookup was hard to
follow, not the data.

**The fix is one token in the `.blf` — no binary patch at all:**

```diff
-Processor_Type = PXA1802
+Processor_Type = PXA1826
```

---

## 1. What the gate really is

`CWtptpDownLoad::GetDeviceBootType` @ `0x081361F0` does **not** determine the
platform. It *dispatches* on a value that was decided long before, at BLF-parse
time. The chain, all reads verified:

| # | where | code |
|---|---|---|
| 1 | BLF text | `[TIM_Configuration]` → `Processor_Type = PXA1826` |
| 2 | `CBlfParser::GetPlatFormType` @ `0x0808D0E0` | compares the string against SoC names, writes `ePlatForm` |
| 3 | `CIFL_SWD::ProcessBLF` @ `0x080EF130` | `080ef25c: add eax,0x3d70` → `GetPlatFormType(blf, &this+0x3D70)` |
| 4 | `CIFL_SWD::Initilizeparameter` | `080d17d8: mov eax,[edx+0x3d70]` … `080d17eb: mov [ecx+0x24],eax` → `_InstanceParams+0x24` |
| 5 | `CWtptpDownLoad::InitializeBL` @ `0x0813AA90` | `0813abd7: mov eax,[esi+0x24]` … `0813abda: mov [ebx+0x1c4c],eax` |
| 6 | `GetDeviceBootType` | `081369fd: cmp DWORD PTR [esi+0x1c4c],0x12` / `jbe` → `08136a36: jmp DWORD PTR [eax*4+0x083371DC]` |

Step 5 is the whole story:

```asm
0813abce  mov eax,[esi+0x30]
0813abd1  mov [ebx+0x1c04],eax      /* _InstanceParams+0x30 -> this+0x1C04 */
0813abd7  mov eax,[esi+0x24]        /* _InstanceParams+0x24 = ePlatForm */
0813abda  mov [ebx+0x1c4c],eax      /* this->m_ePlatform                 */
```

So **the BLF chooses the branch, and the branch checks the board's identity.**

## 2. The 19-entry dispatch table

`0x083371DC`, 19 × 4 bytes. 15 distinct handlers:

| enum | target | | enum | target |
|---|---|---|---|---|
| 0, 6, 12 | `0x08136A06` | **"Unknown platform type" stub** | 8 | `0x08136CD8` |
| 1 | `0x08136E33` | | 9, 11 | `0x08136C91` |
| 2 | `0x08136DF4` | | 10 | `0x08136C4A` |
| 3 | `0x08136DAD` | | 13 | `0x08136C03` |
| 4, 7 | `0x08136D66` | | 14 | `0x08136BBC` |
| 5 | `0x08136D1F` | | 15 | `0x08136B75` ← **ours** |
| | | | 16 | `0x08136B2E` |
| | | | 17 | `0x08136AE7` |
| | | | 18 | `0x08136A3D` |

## 3. What each handler checks

`T.1438` @ `0x081323D0` is not a table lookup — it is just

```c
size_t T.1438(std::string& haystack, const std::string& needle) {
    return haystack.find(needle.c_str(), 0, needle.size());
}
```

Every handler performs three substring tests and, on success, falls into the
shared tail at `0x08136A97`:

```c
SetBootDeviceType(dev, true);
return 1;                      /* 0x08136A97 / 0x08136ADD */
```

The three subjects come from the `GetVersion` (opcode `0x20`) reply:

| variable | built from | sample on our board |
|---|---|---|
| `[ebp-0x4488]` processor | `resp[14..17]` as `%c%c%c%c` | `NZA3` |
| `[ebp-0x448c]` version    | `resp[6..9]` as `%c%c%c%c`   | `3311` |
| `[ebp-0x4490]` date        | `resp[18..21]` as `%02x%02x%02x%02x` | `06012014` |

(`resp[10..13]` is the same date as a little-endian `u32`, printed `%08X` —
the protocol carries the date twice, in LE word form and in digit order.)

`0x08136B75`, enum 15, verbatim:

```asm
mov eax,[ebp-0x449c]          ; date string
mov edx,0x8406848             ; "06012014"
call 0x81323d0                ; T.1438
cmp eax,0xffffffff
je   fail
mov edx,0x8406840             ; "NZA3"
lea eax,[ebp-0x4488]          ; processor
call 0x81323d0
cmp eax,0xffffffff
je   fail
mov eax,[ebp-0x44a0]          ; version
mov edx,0x8406844             ; "3311"
call 0x81323d0
jmp  0x08136a7f               ; -> SetBootDeviceType(dev,true)
```

**Our board reports Version `3311`, Date `06012014`, Processor `NZA3` — all
three match enum 15.** This is a direct three-way match, not a guess.

## 4. The complete identity table

The lookup keys are `std::string` globals at `0x08406800…`, all constructed
from `.rodata` literals by `_GLOBAL__I__…CWtptpDownLoad` @ `0x0812FC40`.
Full mapping, enum → (date, processor, version):

| enum | date | processor | version | platform |
|---|---|---|---|---|
| 0, 6, 12 | — | — | — | **dead — "Unknown platform type"** |
| 1 | `06242009` | `TVTD` | `3211` | |
| 2 | `06242009` | `TVTD` | | |
| 3 | | | | |
| 4, 7 | `08182011` | | | |
| 5 | `03082012` | `MMP3` | `3302` | |
| 8 | `07022012` | `EMEI` | `3304` | |
| 9, 11 | `01152013` | `HELN` | `3307` | |
| 10 | `04162013` | `EDEN` | `3307` | |
| 13 | `02122014` | `HLN2` | `3308` | |
| 14 | `02142014` | `EDEN` | `3307` | |
| **15** | **`06012014`** | **`NZA3`** | **`3311`** | **NEZHA3 — our board** |
| 16 | `09172014` | `ULC1` | `3313` | |
| 17 | `11052014` | `HLN3` | `3313` | |
| 18 | `25032015` | `HLN4` | `3313` | |

Note `NZA3` appears at exactly one place — the entry for `3311` /
`06012014`. The board's own TIM header at `mtd0` image offset `+0x24` holds
`0x33333131`, `0x06012014`, `0x4E5A4133`, which is the same triple.

## 5. `Processor_Type` → `ePlatForm`

`GetPlatFormType` is a linear `std::string::compare` chain. The entries that
matter, read straight from the code:

```asm
; PXA1802
0808d258  mov  DWORD PTR [esp+0x4],"PXA1802"
0808d260  mov  DWORD PTR [esp],ebx
0808d263  call std::string::compare
0808d268  test eax,eax
0808d26a  jne  0808d2f8
0808d270  mov  edx,DWORD PTR [ebp-0x82c]     ; &out
0808d276  mov  DWORD PTR [edx],0x6          ; ePlatForm = 6   <-- DEAD SLOT
0808d27c  mov  DWORD PTR [esp+0x4],"NEZHA platform type\n"

; PXA1826
0808d48f  mov  DWORD PTR [esp+0x4],"PXA1826"
0808d497  mov  DWORD PTR [esp],ebx
0808d49a  call std::string::compare
0808d49f  test eax,eax
0808d4a1  jne  0808d509
0808d4a3  mov  edx,DWORD PTR [ebp-0x82c]
0808d4a9  mov  DWORD PTR [edx],0xf          ; ePlatForm = 15  <-- LIVE
0808d4af  mov  DWORD PTR [esp+0x4],"NEZHA3 platform type\n"
```

Recovered subset of the full chain:

| ePlatForm | `Processor_Type` | platform name printed |
|---|---|---|
| 3 | `PXA1202` | |
| **6** | **`PXA1802`** | `NEZHA platform type` |
| 9 | `PXA1088` | |
| 11 | `PXA1920` | |
| 13 | `PXA1U88` | |
| **15** | **`PXA1826`** | `NEZHA3 platform type` |
| 17 | `PXA1936` | |

**`PXA1802` → enum 6 → the dead stub.** That is the entire failure. Every BLF
in this repo declared `PXA1802`.

`PXA1826` is not merely a workaround: the board's own processor string is
`NZA3`, i.e. **Nezha 3**, and `PXA1802` is plain **Nezha**. The BLF was
mis-declaring the SoC family.

## 6. What to change

Every `.blf` in the repo carries `Processor_Type = PXA1802` and must carry
`PXA1826`:

```
findings/tooling/blf/p298_fixed.blf
findings/tooling/blf/p298_probe.blf
tools/mk-mtd21-blf.py                  (generator — fix the template)
flashing_tools/p298-swd/*.blf
flashing_tools/wine-swd/*.blf
```

With the BLF corrected, the platform-enum patch
(`tools/swdl-platform-patch.py`) becomes unnecessary. If it is still wanted as a
fallback, the correct constant is **`--enum 0x0F`**, not `0x12`. `0x12` selects
the HLN4 / `3313` / `25032015` branch, which does not match this board.

## 7. Offline confirmation with the vendor oracle

`MakeDownloadPackage` (the vendor's own Linux binary, same parser, no USB and
no root) was run on the probe BLF before and after the edit:

```console
$ LC_ALL=C LANG=C Utilities/MakeDownloadPackage/MakeDownloadPackage \
      -r old_probe.blf -o old.zip | grep platform
NEZHA platform type
NEZHA platform type

$ LC_ALL=C LANG=C Utilities/MakeDownloadPackage/MakeDownloadPackage \
      -r p298_probe.blf -o new.zip | grep platform
NEZHA3 platform type
NEZHA3 platform type
```

The `NEZHA3 platform type` string lives at `.rodata` `0x083302EF` and is
emitted by the `PXA1826` arm of `GetPlatFormType` (`0x0808D4AF`).

**What this does and does not prove.** It confirms that `GetPlatFormType`
reads `Processor_Type` from the BLF and that `PXA1826` reaches the `ePlatForm
= 15` arm. It does **not** validate the gate itself:
`MakeDownloadPackage` builds the download package and exits, and never calls
`CWtptpDownLoad::GetDeviceBootType`, which is where the enum is dispatched.
Measured: the oracle reports `Make Download Package successfully` for the
`PXA1802` BLF too. So a passing oracle is a necessary condition, not a
sufficient one.

The gate analysis in §1–§5 stands on the disassembly alone, which is
unambiguous: five instruction reads from `GetPlatFormType` through
`Initilizeparameter` and `InitializeBL` to the jump table, plus the three
string literals at `0x08406840/44/48`. Only a live run against the board can
confirm the transfer.

## 8. Risk to check before a real flash

`Processor_Type` is not only read by `GetDeviceBootType`. `CIFL_SWD::GenerateTim`
passes the same `ePlatForm` to
`CImageBuilder::BuildBinFile(CTimDescriptor&, eBLFVersion, ePlatForm, …)`
(`0x080F2184`), so the generated TIM/FBF may differ between enum 6 and enum 15.
Validate with the vendor's own oracle before writing anything:

```sh
LC_ALL=C LANG=C Utilities/MakeDownloadPackage/MakeDownloadPackage \
    -r <new>.blf -o /tmp/pkg.zip
```

and diff the resulting `FBF_h.bin` against the one from the old BLF. A change
in the TIM is expected; a change in the *image* payload is not.

## 9. Still unknown

* Whether the remaining handlers (1–14, 16–18) differ in behaviour beyond the
  three identity tests — they all converge on the same tail here, so the
  enum appears to select a platform *check*, not a strategy.
* The empty rows in §4 (enums 2, 3, 4/7) — their keys are constructed from a
  different literal group and were not extracted; not needed for this board.
* Whether Marvell shipped a later `SWDownloader` with a populated enum 6.