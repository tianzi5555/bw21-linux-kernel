# BW21 Linux extension status

## User's required SD layout

The user requires Linux to access the *whole existing FAT32 partition* on the removable SD card, not a disk-image file. The intended guest mount is `/dev/vda1` (virtio-blk over a raw SD-sector backend), mounted e.g. at `/mnt/sd`.

## Verified

- Repository: `tianzi5555/bw21-linux-kernel`.
- GitHub Actions run #2 (`d286c374e81fbe981ca8a1469d9342589cf5642a`) completed successfully in about 24 minutes and uploaded artifact `bw21-linux-image` (2,163,613 bytes). This proves only the configured Buildroot job completed and produced an artifact; it does not prove the image boots with virtio devices.
- BW21 Arduino SDK header exposes `SD_Init`, `SD_DeInit`, `SD_GetCapacity`, `SD_ReadBlocks`, and `SD_WriteBlocks` in `system/component/soc/8735b/app/file_system/drivers/sdio/realtek/sdio_host/inc/sd.h`.
- A minimal AMB82-MINI sketch referencing those API declarations compiled successfully (Arduino CLI exit 0). This is compile-only, not an on-board sector read/write test.
- Existing mini-rv32ima source uses a flat 0x80000000 guest RAM mapping and handles only the UART/CLINT/SYSCON MMIO in its platform wrapper. Its standard machine config/device tree currently has no virtio device or PLIC node.

## Remaining blocking implementation work

- Inspect the *final built kernel config* to confirm Kconfig retained `CONFIG_BLOCK`, `CONFIG_VIRTIO_BLK`, `CONFIG_NET`, `CONFIG_VIRTIO_NET`, `CONFIG_VFAT_FS`, partition-table drivers, and supporting options. Current artifact download/log access is not yet verified.
- Implement standard virtio-mmio transport/device model(s), starting with block, including descriptor/avail/used rings in guest RAM and raw SD sector backend.
- Add external interrupt delivery compatible with the guest kernel's DTB/controller; the existing emulator only raises CLINT timer interrupts. Add matching PLIC/virtio DTB nodes and validate the interrupt numbering/address map.
- Implement virtio-net and a bridge to BW21's already-connected Wi-Fi; Wi-Fi scan/connect remains firmware-controlled and must be exposed to the guest via a defined control path if desired.
- Test on the actual board. Before Linux owns the card, ensure host FatFS is unmounted and never concurrently accesses the card. Firmware flashing requires the user to press the board's hardware boot/download controls; updating `Image` requires user to remove the SD card and write it with a reader.

## Safety

Do not instruct the user to mount/write the card from Linux until virtio block operations and clean handoff have been validated. Raw pass-through can corrupt the FAT volume if FatFS and Linux access it concurrently.
