# BW21 Linux extension status

## User requirement
Linux must access the whole existing FAT32 SD partition directly, not a disk-image file. Intended guest device is `/dev/vda1`, mounted e.g. at `/mnt/sd`.

## Verified facts
- Repo: `tianzi5555/bw21-linux-kernel`.
- GitHub Actions run #2 at commit `d286c374e81fbe981ca8a1469d9342589cf5642a` completed successfully in ~24 minutes and uploaded `bw21-linux-image`, 2,163,613 bytes. This only proves Buildroot produced an Image; the final kernel config has NOT yet been verified and the image is not proven to boot with virtio.
- BW21 SDK header declares `SD_Init`, `SD_DeInit`, `SD_GetCapacity`, `SD_ReadBlocks`, and `SD_WriteBlocks` in `system/component/soc/8735b/app/file_system/drivers/sdio/realtek/sdio_host/inc/sd.h`.
- The current Linux sketch now contains a raw-sector backend wrapper (512-byte sectors, bounds checking, aligned 8-sector bounce buffer, explicit FatFS/raw ownership handoff) using these APIs. It compiles for AMB82-MINI, exit 0; warnings say the wrappers are unused because the virtio layer is not implemented. This is compile-only; no hardware sector I/O test has been done.
- mini-rv32ima currently has a flat guest RAM map and the platform wrapper handles UART, CLINT and SYSCON only. Existing DTB lacks virtio devices and a PLIC/external interrupt controller.

## Local, not yet pushed
- `kernel_fragment.config` now also requests `CONFIG_PARTITION_ADVANCED`, `CONFIG_MSDOS_PARTITION`, and `CONFIG_EFI_PARTITION` so the FAT partition can enumerate as `/dev/vda1`.
- Workflow now includes a final `.config` assertion requiring BLOCK, VIRTIO_BLK, NET, INET, VIRTIO_NET, VIRTIO_MMIO, VFAT_FS, and MSDOS_PARTITION.
- These changes are locally committed as `701205a` plus an uncommitted workflow edit. Push failed because TLS through the configured proxy at `127.0.0.1:7890` is failing. Do not bypass the proxy.

## Remaining engineering work
1. Push corrected kernel config/workflow when proxied TLS works; verify final `.config` and download the artifact.
2. Implement virtio-mmio block transport in mini-rv32ima, descriptor/avail/used queues, SD raw-sector backend, and correct host-FATFS-to-guest handoff (never concurrent access).
3. Add interrupt delivery and matching PLIC/virtio device tree. Current emulator only injects CLINT timer interrupts.
4. Implement virtio-net and bridge guest Ethernet to firmware Wi-Fi; provide a defined firmware control path for Wi-Fi scan/connect (Linux cannot directly control the RTL8735B radio).
5. Compile firmware and test with actual board. User must be explicitly asked to use hardware download mode before each flash; user must remove SD and write new kernel with reader.

## Safety
Do not tell user to mount/write the FAT partition until block I/O and clean handoff are validated; incorrect raw-sector writes can corrupt the filesystem.
