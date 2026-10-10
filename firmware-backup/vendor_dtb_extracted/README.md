# Vendor DTB extraction (kernel-a.bin)

- Source: `ooredoo_backup/kernel-a.bin` (shipping unit, kernel 3.10.33).
- The DTB is appended to the zImage at offset `0x2b9ef0`
  (magic `d0 0d fe ed`, totalsize 0x9430 = 37,936 bytes).
- `vendor.dtb` = raw blob; `vendor.dts` = decompiled (`dtc -I dtb -O dts`).

This is the authoritative hardware description used for the mainline RFC:
SoC = ASR1802S (Cortex-A7, NOT A9), Marvell MMP-derived blocks
(mrvl,mmp-intc / mrvl,mmp-timer / mrvl,mmp-uart / marvell,pdma-1.0 /
mrvl,mmp-twsi), RAM @ 0x04000000 (128 MiB), 25 MTD partitions,
asr,nezhas-mmc, asr,qspi, asr,asr-eth-v2, asr,pcie-kagu, snps,dwc3.
