# Marvell_SWDownloader_v4.8.8.2

Marvell / ASR SWDownloader 4.8.8.2 — **the tool used for all work in this
repository.** The only one of the four with a native Linux binary.

## No binaries are redistributed here

Download from Marvell/ASR or a third-party mirror and verify against the
checksums below.

## Identity

| | |
|---|---|
| product name | `Marvell_SWDownloader_v4.8.8.2` |
| reported version | `SWDL Version: 4.8.8.2` |
| reported build date | `SWDL Date: 2015-8-3` |
| file timestamps | 2015-08-04 |
| unpacked folder size | 123 MB |
| distribution origin | Marvell, via third-party mirrors |

## Files

| file | size | md5 |
|---|---|---|
| `SWDownloader.exe` | 21,754,880 | `bab18d6b3df1b1f494c3f4fb251d7521` |
| `swdl_linux` | 4,622,608 | — |
| `Utilities/IFLSWD/` — `IFL_SWD.dll`, `IFL_SWDAPI.h`, sample sources | | |
| `Utilities/FBFMake/` — `FBFMake`, `mkotafbf`, readmes | | |
| `Utilities/TimBuilder/` — `TimBuilder`, readme | | |
| `Utilities/MakeDownloadPackage/` — `MakeDownloadPackage`, `DownloadZipPackage` | | |
| `Utilities/RomCopy/` — `RomCopy_linux` (NAND not supported) | | |
| `Utilities/RadioImgMaker/`, `Utilities/ReliableData/` | | |
| `Utilities/IFLSWD/IFL_SWD_Sample/` | | `IFLSWD_Sample.cpp` — documented API sample |

### `swdl_linux`

ELF 32-bit LSB, Intel i386, **statically linked**, not stripped, with DWARF
`debug_info`. No 32-bit runtime libraries needed. The debug info is what made
the `.blf` format recoverable — see `../blf/swdownloader-blf-spec.md`.

## Command line

```
usage to do download:        ./swdl_linux -D BLF_filename [-T Timeout(seconds)] [-N]
usage to do single download: ./swdl_linux -D BLF_filename -S [-T Timeout(seconds)] [-N]
usage to do upload:          ./swdl_linux -U Upload_config_filename
```

* `-D` — **write** flash from a `.blf`
* `-U` — **read** flash using a `.cfg` (read-only; needs no `.blf`)
* `-T` — overall timeout in seconds
* `-N`, `-S` — mode flags

## Behaviour notes

* It runs `rmmod wtptp` at start; the `Module wtptp is not currently loaded`
  message is harmless.
* **Needs root.** Without root it exits silently with no banner at all, which
  looks like a silent crash rather than a permission error.
* **Needs `LC_ALL=C LANG=C`**, otherwise it aborts with
  `locale::facet::_S_create_c_locale name not valid`.
* Resolves the `.blf` and image paths relative to **its own directory**, not
  the current working directory. The same is true of the bundled utilities.
* Writes `SWDownLoader.log` in its own directory.

## Bundled Linux utilities (no router, USB or root needed)

```bash
export LC_ALL=C LANG=C
./Utilities/TimBuilder/TimBuilder            -r BLF_filename [-f tim_folder]
./Utilities/MakeDownloadPackage/MakeDownloadPackage -r BLF_filename -o package.zip
./Utilities/FBFMake/FBFMake                  -r BLF_filename -f fbf_folder
./Utilities/RomCopy/RomCopy_linux            -b BLF_filename -o out.bin
```

These run the same `CBlfParser` as the main tool and are the way to validate a
`.blf` before touching hardware. `MakeDownloadPackage` reproduces the full DKB
failure chain and is the faithful oracle.

## Status against this board

* `.blf` parsing: **solved** — see `../../blf/flash_recovery.blf`
* flash read (`-U`): implemented, config format partly recovered, **not yet
  verified against hardware**
* flash write (`-D`): **blocked** — `OBMImage is NULL in prepareDKB list`;
  the DKB stage needs `WTMI` and `TZII`, which are not obtainable

## Related

* `.blf` format spec: `../blf/swdownloader-blf-spec.md`
* Working configs: `../../blf/`
* Sibling tools and why they do not help: `../README.md`
