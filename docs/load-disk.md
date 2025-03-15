---
id: load-disk
title: Loading Disk Blocks to Memory
sidebar_label: Loading Disk Blocks to Memory
---

As we progress through the stages, our Operating System will cease to fit in 512 bytes. Hence we'll need to write other pieces of our code in different disk blocks. To be able to use them in our memory, we'll have to load them first from the disk to memory. This is exactly what we shall be doing in this chapter.

Let us recall the structure and working of the disk once again. In XSM, the disk was a software simulation and where disk reads were done using a single disk head and the disk was a long array of bytes. However actual hard disks consists of multiple disk platters layered on top of one another on a spindle. Each of these disks have two disk read heads to read from it one for each - top and bottom surfaces. On a lower level, each disk is divided into tracks and sectors. Tracks are concentric circular rings on the disk and each of the track is further divided into segments called sectors. The farthest track from the spindle is numbered as 0. The second farthest one is track 1 and so on. A collection of the same numbered tracks on all the disks is called a cylinder. 

For reading blocks from the disk, we will be using a different BIOS interrupt, INT 0x13 in particular. This interrupt contains service functions offered by the BIOS for handling diskette operations. We are interested in the disk read service and this is specified in the `ah` register (as seen earlier in print_string). The number of sectors to be read is given in `al`. The cylinder number (track number) is to be specified in `ch` and the sector to start reading from in `cl`. `dl` stores the drive number (0 for Floppy1, 0x80 for HDD1 etc). We need not set this explicitly as BIOS automatically sets this to the appropriate disk upon calling the interrupt if `dl` is not set by us. Finally `dh` stores the head number (platter number and which surface) to read from. After the interrupt is called, the data is loaded in the same order starting from address `[es:bx]`. Hence we specify `es` and `bx` with the segment and memory offset we want to load the blocks into.

We will now write the actual function for loading blocks from the disk to memory. We shall write this in a new file and assume that only the number of sectors to be read, memory segment and memory offset to load this to as variables. We shall assume that the caller of this function already sets this in the `dh`, `es` and `bx` registers respectively.

<p class="codeblock-label">boot/disk.asm</p>

```
disk_load:
	pusha ; Save all registers
	mov ah, 0x02 ; Choose the "disk read" function
	mov al, dh ; Number of sectors to read
	mov ch, 0x0 ; Cylinder number (0 is the outermost one)
	mov cl, 0x02 ; Start reading from second sector. Index starts from 1
	push dx ; Save dx as we will update dh next
	mov dh, 0x0 ; Read from head 0
	int 0x13 ; Call BIOS diskette interrupt

	pop dx ; Restore dx so that dh contains the number of sectors again

	jc disk_load_error
	cmp al, dh
	jne sectors_error

	popa
	ret

disk_load_error:
	mov bx, DISK_LOAD_ERROR
	call print_string

sectors_error:
	mov bx, SECTORS_ERROR
	call print_string

DISK_LOAD_ERROR: db "Error loading from Disk", 0
SECTORS_ERROR: db "Incorrect numbers of sectors read", 0
```

Modify our boot sector code (`boot/boot.asm`) to include `boot/disk.asm` functions in it.

<p class="codeblock-label">boot/boot.asm</p>

```
[org 0x7c00]

mov bx, MSG_REAL_MODE
call print_string
jmp $

%include "boot/print_string.asm"
%include "boot/disk.asm"

MSG_REAL_MODE db "Entered 16 bit Real Mode", 0

times 510-($-$$) db 0
dw 0xaa55
```