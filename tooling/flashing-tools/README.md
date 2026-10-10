# Marvell / ASR downloader tools — inventory and comparison

Four flashing products were evaluated against this board. **None of the
binaries are redistributed**; each subdirectory records the exact product
name, version, build date, file sizes and checksums so the same tool can be
identified and verified elsewhere.

All four are proprietary Marvell / ASR / third-party redistributions of the
same core. They are dated 2012–2015 and are not affiliated with this project.

---

## Comparison

| | `Marvell_SWDownloader_v4.8.8.2` | `Marvell_SwUpgrade_P_V1.05` | `Marvell_SwUpgrade_C_v1.03` | `ONTIM_Marvell_MultiDL_v1.4.3.72` |
|---|---|---|---|---|
| build date | **2015-08-04** | 2014-12-03 | 2015-06-25 | 2014-06-15 |
| folder size | 123 MB | 23 MB | 24 MB | 13 MB |
| main executable | `SWDownloader.exe` 21,754,880 B | `Marvell_SwUpgrade_P_V1.05.exe` 10,687,488 B | `Marvell_SwUpgrade_C.exe` 15,316,992 B | `ONTIM_Marvell_MultiDL.exe` 1,485,824 B |
| **native Linux binary** | **yes — `swdl_linux` 4,622,608 B** | no | no | no |
| GUI required | optional (CLI mode) | yes | yes | yes |
| login protection | no | **yes — `Password.txt`: `SHHIPAD`** | no | no |
| supports `PXA1802` | yes | yes | yes | yes |
| `.blf` grammar | `[TIM_Configuration]` / `[Flash_Properties]` / `[Image_List]` | identical | identical | identical |
| flash-read (`-U` / dump) | yes | yes | yes | yes |
| needs `WTMI`+`TZII` to write | yes | yes | yes | yes |

The OBM in `nand-dumps/mtd0.bin` carries the banner
`SWDownloader 4.9.0.5 for %s`, i.e. it was built for tool version 4.9.0.5.
**None of the four is 4.9.** `Marvell_SWDownloader_v4.8.8.2` is the newest of
the set and is the only one that runs natively on Linux, so it is the one all
work was done with.

## All four share one core library

Each product ships `IFL_SWD.dll` — the same library that `swdl_linux` links
statically. All three DLLs (all different builds) contain, as UTF-16LE:

```
OBMImage is NULL in prepareDKB list
ERROR in Prepare %s DKBlist to be download into BootRom! NO OBMI in image list
    or not Appoint FlasherLoadAddress in blf
OBMI   TIMH   WTMI   TZII
TIM_Configuration   Flash_Properties   Image_List   %d_Image_%s
Processor_Type   Flash_Family   SPI-NAND
PXA1802   NEZHA platform type
PreforUpload....   Area_UploadData_SaveFile   Finish to dump to fbf file
```

Consequences:

* The `.blf` format is the same in all four — see `../blf/swdownloader-blf-spec.md`.
* The write path is blocked in all four, for the same reason: the DKB stage
  needs `WTMI` and `TZII`, which are not part of any MTD partition and are not
  obtainable. **No alternative product bypasses this.**
* All four also implement a **flash read** path that needs none of those
  blobs. That is the useful capability here.

## Requirements for any of them

* Device must be in ASR ROM boot mode; USB enumerates as `2ecc:3002`
  (product string `KSTR`, manufacturer `ASR`). The device drops the USB port
  after ~1.5 s waiting for a host handshake, so start the tool *first*, then
  power-cycle or assert the `USB_BOOT` test point.
* Windows builds need the Marvell USB driver (`Driver/Download.url` inside
  each folder points at the download page).
* All builds predate modern locale handling: run under `LC_ALL=C LANG=C` or
  they abort with `locale::facet::_S_create_c_locale name not valid`.
* `swdl_linux` additionally needs root — it runs `rmmod wtptp` and talks to
  USB. Without root it exits silently with no banner.

## Security note

`Marvell_SwUpgrade_P_V1.05` ships a `Password.txt` containing a login
credential (`SHHIPAD`). It is recorded here only because it is part of the
tool's own distribution and is needed to evaluate the tool. Do not use it on
any system you care about; these are unsigned 2012–2015 binaries.

## Subdirectories

| directory | product |
|---|---|
| `Marvell_SWDownloader_v4.8.8.2/` | Marvell SWDownloader, the only one with a Linux CLI |
| `Marvell_SwUpgrade_P_V1.05/`     | Marvell SwUpgrade P — login-locked |
| `Marvell_SwUpgrade_C_v1.03/`     | Marvell SwUpgrade C |
| `ONTIM_Marvell_MultiDL_v1.4.3.72/` | ONTIM Marvell MultiDL — smallest, redistributable third-party build |
