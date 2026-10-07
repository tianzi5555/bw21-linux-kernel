# BW21-CBV-Kit nommu Linux — current status

## Required target
Linux must access the entire existing FAT32 SD partition directly (no disk image), expected as `/dev/vda1` mounted at `/mnt/sd`.

## Verified
- Repo `tianzi5555/bw21-linux-kernel` exists.
- GitHub Actions run #2 at commit `d286c374e81fbe981ca8a1469d9342589cf5642a` succeeded and uploaded a 2,163,613-byte `bw21-linux-image` artifact. Final config was not inspected, and this image has not been shown to boot with virtio.
- BW21 SDK declares `SD_Init`, `SD_DeInit`, `SD_GetCapacity`, `SD_ReadBlocks`, and `SD_WriteBlocks` in the RTL8735B SDIO `sd.h`.
- Current Arduino sketch has a bounds-checked 512-byte raw SD sector backend wrapper, aligned 8-sector bounce buffer, and guards against FatFS/raw ownership overlap. It compiles for AMB82-MINI, exit 0. Functions still warn unused because no virtio device calls them. No board sector-I/O test.
- mini-rv32ima platform wrapper currently emulates only UART, CLINT and SYSCON MMIO; current DTB has no virtio node/PLIC. Local `mini-rv32ima.h` now has an optional `MINIRV32_EXTERNAL_IRQ_PENDING(state)` hook that sets `mip.MEIP`, wakes WFI, and prioritizes MEIP before MTIP. AMB82 sketch compiles with the hook defaulting false (Arduino CLI EXIT 0), so existing behavior is unchanged. No PLIC source or runtime MEIP test exists yet.

## Local commits not pushed
- Commit `701205a` adds MBR/GPT partition config and status notes.
- Commit `1dc69b0` adds kernel final-config assertions; workflow+fragment changes include block/net/FAT options.
- Proxy at `127.0.0.1:7890` accepts TCP but GitHub TLS via Schannel/OpenSSL fails with EOF. Do not bypass proxy; push is blocked until its routing/TLS is fixed.

## Remaining
1. Push local config/workflow fixes through the required proxy; verify final `.config` and retrieve the artifact.
2. Implement virtio-mmio block registers/queues/descriptor chains and connect them to raw SD sector functions.
3. Implement external interrupt delivery/PLIC and matching DTB nodes.
4. Implement virtio-net plus firmware Wi-Fi bridge and scan/connect control path.
5. Compile, then test on hardware. User must manually enter hardware download mode before each firmware flash, and remove SD/write Image with card reader. Never have FatFS and guest raw block I/O active simultaneously.
6. Validate a FAT32 mount/read/write round trip and network ping before reporting done.

## Safety
Do not tell user to mount/write SD from Linux until virtio block, partition enumeration, and ownership handoff are validated. The currently compiled firmware is not ready to flash for this feature.
