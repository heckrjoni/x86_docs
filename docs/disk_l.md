---
id: disk_l
title: Disk Load Implementation
sidebar_label: Disk load in x86
---
## Disk Load Implementation

To understand the disk type and retrieve details such as the number of sectors, cylinders, and more, we utilized the following QEMU commands:

```
qemu-system-i386 -drive file=os-image.bin,format=raw,if=none,id=mydisk -device ide-hd,drive=mydisk,bus=ide.0 -monitor stdio

info block

info qtree
```

From this analysis, we confirmed that QEMU provides an IDE disk interface.

### Implementing a Basic Disk I/O Function

Based on our findings, we implemented a simple disk I/O function. However, handling disk operations and hard drive interactions is inherently complex. The exact commands and operations vary across devices. Therefore, we decided that the best approach would be to analyze how QEMU's BIOS performs disk I/O when loading the kernel during the boot phase.

To achieve this, we used:

```
qemu-system-i386 -drive file=os-image.bin,if=ide,format=raw -d trace:ide*
```

By examining the trace output, we understood how QEMU implements IDE disk I/O. Using this insight, we implemented our own disk I/O operation that takes the sector number and number of sectors to read as input.

### Assembly Implementation

Below is the assembly code used for performing disk read operations:

```
; Read from device control register (0x3F6)
mov dx, 0x3F6
in al, dx

; Read from status register (0x1F7)
mov dx, 0x1F7
in al, dx

; Disable interrupts by writing to 0x3F6
mov dx, 0x3F6
mov al, 0x08
out dx, al

; Modify control register (0x3F6)
mov dx, 0x3F6
mov al, 0x0A
out dx, al

; Read status and drive/head register
mov dx, 0x1F7
in al, dx

mov dx, 0x1F6
in al, dx

; Select drive and enable LBA mode
mov dx, 0x1F6
mov al, 0xE0   ; Master drive, LBA mode, Head 0
out dx, al

; Feature register (not used in basic read)
mov dx, 0x1F1
mov al, 0x00
out dx, al

; Set number of sectors to read
mov dx, 0x1F2
mov al, 0x02  ; Read 2 sectors
out dx, al

; Set LBA address (sector number = 0x64)
mov dx, 0x1F3
mov al, 0x64
out dx, al

mov dx, 0x1F4
mov al, 0x00
out dx, al

mov dx, 0x1F5
mov al, 0x00
out dx, al

; Issue READ SECTORS command (0x20)
mov dx, 0x1F7
mov al, 0x20
out dx, al

; Read status register
mov dx, 0x1F7
in al, dx

; Read 512 bytes (256 words) from disk
mov cx, 512
mov dx, 0x1F0
mov edi, [freePageByte]  ; Destination memory address (2GB offset)

read_loop:
    in ax, dx    ; Read 16-bit word
    mov [edi], ax
    add edi, 2   ; Move to the next word
    loop read_loop

ret

```

### Explanation

The disk data is read from port `0x1F0` as 16-bit words and stored in memory at `freePageByte`, which is allocated at the 2GB mark. This is because the first 2GB is reserved for the kernel, while the remaining space is allocated for user programs. The `read_loop` copies the data efficiently.

### Running QEMU with Required Memory

To ensure successful execution, QEMU must be run with increased memory, as its default allocation of 128MB is insufficient for accessing memory beyond 2GB. Use the following command:

```
qemu-system-i386 -s -S -m 4096M -drive file=os-image.bin,format=raw
```

Here, `-m 4096M` allocates 4GB of RAM, ensuring that the disk read operation can access the designated memory regions without issues.


















Detailed Explanation of IDE Disk I/O in Assembly

Understanding IDE Disk I/O Ports and Commands

Step-by-Step Breakdown of the Assembly Code
1. Checking Device Control Register

mov dx, 0x3F6
in al, dx

Reads from device control register (0x3F6) to check if the disk is busy or has pending operations.
2. Checking Drive Status

mov dx, 0x1F7
in al, dx

Reads the status register (0x1F7), which contains flags like:

    BSY (Busy) - 1 if the disk is busy.
    DRDY (Drive Ready) - 1 if the disk is ready for commands.
    DRQ (Data Request) - 1 if the disk is ready to transfer data.

3. Disabling and Enabling Interrupts

mov dx, 0x3F6
mov al, 0x08
out dx, al

    Writes 0x08 to device control register (0x3F6) to disable drive interrupts.

mov dx, 0x3F6
mov al, 0x0A
out dx, al

    Writes 0x0A to device control register (0x3F6) to modify the control settings.

4. Selecting the Drive and Enabling LBA Mode

mov dx, 0x1F6
mov al, 0xE0
out dx, al

    Writes 0xE0 to Drive/Head Register (0x1F6) to select:
        Master drive
        LBA mode enabled
        Head 0 selected

5. Setting Up the Read Operation

mov dx, 0x1F1
mov al, 0x00
out dx, al

    Writes 0x00 to the features register (0x1F1) (not used in basic reads but required for proper setup).

mov dx, 0x1F2
mov al, 0x02
out dx, al

    Writes 0x02 to sector count register (0x1F2) to read 2 sectors (1024 bytes).

mov dx, 0x1F3
mov al, 0x64
out dx, al

    Writes 0x64 to LBA low byte register (0x1F3) (first 8 bits of sector number).

mov dx, 0x1F4
mov al, 0x00
out dx, al

    Writes 0x00 to LBA mid byte register (0x1F4) (middle 8 bits of sector number).

mov dx, 0x1F5
mov al, 0x00
out dx, al

    Writes 0x00 to LBA high byte register (0x1F5) (high 8 bits of sector number).

mov dx, 0x1F7
mov al, 0x20
out dx, al

    Writes 0x20 to command register (0x1F7) to issue the READ SECTORS (0x20) command.

6. Waiting for Data to be Ready

mov dx, 0x1F7
in al, dx

    Reads the status register (0x1F7) again to check if data is ready.
    Waits for Bit 3 (DRQ - Data Request) to be set before reading.

7. Reading Data from the Disk

mov cx, 512
mov dx, 0x1F0
mov edi, [freePageByte]

    Sets up a loop to read 512 bytes (1 sector).
    0x1F0 is the data register where data is read.

read_loop:
    in ax, dx
    mov [edi], ax
    add edi, 2
    loop read_loop

    Reads a 16-bit (2-byte) word from port 0x1F0.
    Stores the word at the memory location freePageByte (starting at 2GB).
    Moves the memory pointer forward by 2 bytes.
    Repeats 256 times to read a full 512-byte sector.
