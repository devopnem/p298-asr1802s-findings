# ONTIM_Marvell_MultiDL_v1.4.3.72

ONTIM Marvell MultiDL v1.4.3.72 — third-party GUI flashing tool, smallest of
the four. **No binaries are redistributed here.**

## Identity

| | |
|---|---|
| product name | `ONTIM_Marvell_MultiDL_v1.4.3.72` |
| file timestamps | 2014-06-15 |
| unpacked folder size | 13 MB |
| distribution origin | third-party (ONTIM), repack of the Marvell core |

## Files

| file | size | md5 |
|---|---|---|
| `ONTIM_Marvell_MultiDL.exe` | 1,485,824 | `da57770fec72c84bfcaca496382e8c3e` |
| `IFL_SWD.dll` | 4,407,808 | `e814398e6624521c869ecbe51f5d6d9e` |
| `WtptpDownLoad.dll` | 5,114,880 | `c3fab5e4072e0a327b8ff2b59607d8d1` |
| `IFLWrapper.dll` | 1,878,016 | |
| `Driver/Download.url`, `How to Use.url`, `Credits.txt` | | |

Not login-locked.

## Analysis

* Same Marvell `IFL_SWD` core: same `.blf` grammar, same
  `OBMI` / `TIMH` / `WTMI` / `TZII` DKB requirement, same
  `OBMImage is NULL in prepareDKB list` error string.
* Only build of the four that does **not** contain the string `TwoStag`, which
  corresponds to the `CIFL_SWD::PrepareTwoStagDKBlist()` path where our write
  attempt stops. Its `IFL_SWD.dll` is nevertheless 4.4 MB and still has the
  full `TZII` handling, so this is almost certainly a naming difference rather
  than a different code path — and it is also the oldest build, so it is the
  least likely to match the OBM 4.9.0.0 in `nand-dumps/mtd0.bin`.
* Contains `PreforUpload....`, `Area_UploadData_SaveFile`,
  `Finish to dump to fbf file` → **has the flash-read capability**.
* GUI only.

## Status against this board

| capability | result |
|---|---|
| `.blf` parsing | same format |
| flash write | **blocked** — same missing `WTMI` / `TZII` |
| flash read | present, GUI-driven |

Lowest value of the four: oldest build, GUI only, no CLI.

## Related

* Comparison with the other three tools: `../README.md`
* `.blf` format spec: `../../blf/swdownloader-blf-spec.md`
