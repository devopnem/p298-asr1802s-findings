# Marvell_SwUpgrade_C_v1.03

Marvell SwUpgrade **C** v1.03 — GUI-only Windows flashing tool.
**No binaries are redistributed here.**

## Identity

| | |
|---|---|
| product name | `Marvell_SwUpgrade_C_v1.03` |
| file timestamps | 2015-06-25 |
| unpacked folder size | 24 MB |
| distribution origin | Marvell, via third-party mirrors |

## Files

| file | size | md5 |
|---|---|---|
| `Marvell_SwUpgrade_C.exe` | 15,316,992 | `7fc052363672adcbc9c4c82e9e8ed36a` |
| `IFL_SWD.dll` | 3,896,832 | `202be606c022636df7d37da14d659846` |
| `WtptpDownLoad.dll` | 2,703,872 | `cfffff74112392efa549f0c4191813a7` |
| `GdiPlus.dll` | 1,723,392 | |
| `7z.exe` / `7z.dll` | 163,840 / | `a51d90f2f9394f5ea0a3acae3bd2b219` |
| `Marvell Driver/Download.url`, `How to Use.url`, `Credits.txt` | | |

Not login-locked.

## Analysis

* Same Marvell `IFL_SWD` core: same `.blf` grammar, same four-stage DKB
  (`OBMI`, `TIMH`, `WTMI`, `TZII`), same
  `OBMImage is NULL in prepareDKB list` error string.
* Its `IFL_SWD.dll` contains the widest platform table of the three Windows
  builds — `NEZHA`, `NEZHA2`, `NEZHA3` alongside `PXA1802` and
  `NEZHA platform type`.
* Contains `PreforUpload....`, `Area_UploadData_SaveFile`,
  `Finish to dump to fbf file` → **has the flash-read capability**.
* Contains `Open file %s Failed when Save BLF file ... we can't save your
  modification for blf` → the GUI can **edit and re-save a `.blf`**, which is
  the one thing the Linux build cannot do.
* GUI only.

## Status against this board

| capability | result |
|---|---|
| `.blf` parsing / editing | same format; GUI can save a modified `.blf` |
| flash write | **blocked** — same missing `WTMI` / `TZII` |
| flash read | present, GUI-driven |

The newest of the three Windows builds, but still older than
`Marvell_SWDownloader_v4.8.8.2` and without a CLI.

## Related

* Comparison with the other three tools: `../README.md`
* `.blf` format spec: `../../blf/swdownloader-blf-spec.md`
