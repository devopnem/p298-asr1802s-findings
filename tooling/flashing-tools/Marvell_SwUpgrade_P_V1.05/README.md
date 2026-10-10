# Marvell_SwUpgrade_P_V1.05

Marvell SwUpgrade **P** v1.05 — GUI-only Windows flashing tool.
**No binaries are redistributed here.**

## Identity

| | |
|---|---|
| product name | `Marvell_SwUpgrade_P_V1.05` |
| file timestamps | 2014-12-03 |
| unpacked folder size | 23 MB |
| distribution origin | Marvell, via third-party mirrors |

## Files

| file | size | md5 |
|---|---|---|
| `Marvell_SwUpgrade_P_V1.05.exe` | 10,687,488 | `155578db44e4e0b6b3b247e20e905290` |
| `IFL_SWD.dll` | 4,545,536 | `d595d60722c5d3367639b58eeea18e11` |
| `WtptpDownLoad.dll` | 5,563,904 | `6dcbaaa8b9bd4296f1e240bc73a4b607` |
| `GdiPlus.dll` | 1,723,392 | |
| `7z.exe` / `7z.dll` | 163,840 / | `a51d90f2f9394f5ea0a3acae3bd2b219` |
| `Password.txt` | 23 | contains a login credential |
| `Marvell Driver/Download.url`, `How to Use.url` | | |

## ⚠ Login-locked

`Password.txt` in the distribution contains:

```
Password to Login: SHHIPAD
```

The tool will not start without it. This is an unsigned 2014 binary — run it
only in a throwaway environment, never on a machine you care about.

## Analysis

* Shares the Marvell `IFL_SWD` core: same `.blf` grammar, same
  `OBMI` / `TIMH` / `WTMI` / `TZII` DKB requirement, same
  `OBMImage is NULL in prepareDKB list` error string.
* Contains `PreforUpload....`, `Area_UploadData_SaveFile`,
  `Finish to dump to fbf file` → **has the flash-read capability**.
* Supports `PXA1802` (its `IFL_SWD.dll` contains `PXA1802` and
  `NEZHA platform type` as UTF-16LE).
* GUI only — no command-line mode.

## Status against this board

| capability | result |
|---|---|
| `.blf` parsing | same format as the others |
| flash write | **blocked** — same missing `WTMI` / `TZII` |
| flash read | present, GUI-driven |

Older than `Marvell_SWDownloader_v4.8.8.2` and offers a GUI rather than a CLI,
so it adds nothing except a login prompt.

## Related

* Comparison with the other three tools: `../README.md`
* `.blf` format spec: `../../blf/swdownloader-blf-spec.md`
