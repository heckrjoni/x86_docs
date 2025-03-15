---
id: Kernel
title: Writing our Kernel
sidebar_label: Writing our Kernel and Loading
---
# Writing our Kernel

The involved steps are as follows:
- Write and compile the kernel code.
- Write and assemble the boot sector code.
- Create a kernel image that includes not only our boot sector but our compiled kernel code.
- Load our kernel code into memory.
- Switch to 32-bit protected mode.
- Begin executing our kernel code.

The main function of our kernel merely is to let us know it has been successfully loaded and executed. We can elaborate on the kernel later, so it is important initially to keep things simple.

## Kernel Code

Save the following code in a file called `kernel.c`:

```c
void main () {
    // Create a pointer to a char, and point it to the first text cell of
    // video memory (i.e., the top-left of the screen).
    char *video_memory = (char *) 0xb8000;
    // At the address pointed to by video_memory, store the character 'X'
    // (i.e., display 'X' in the top-left of the screen).
    *video_memory = 'X';
}
```

To compile and link the kernel code:

```bash
$ gcc -ffreestanding -c kernel.c -o kernel.o
$ ld -o kernel.bin -Ttext 0x1000 kernel.o --oformat binary
```

Note that, now, we tell the linker that the origin of our code once we load it into memory will be `0x1000`, so it knows to offset local address references from this origin, just like we use `[org 0x7c00]` in our boot sector, because that is where BIOS loads and then begins to execute it.

## Creating a Boot Sector to Bootstrap our Kernel

We are going to write a boot sector that must bootstrap (i.e., load and begin executing) our kernel from the disk.

Since the kernel was compiled as 32-bit instructions, we are going to have to switch into 32-bit protected mode before executing the kernel code. After we switch into protected mode, the lack of BIOS will make it hard for us to use the disk: we would have to write a floppy or hard disk driver ourselves!

To simplify the problem of which disk and from which sectors to load the kernel code, the boot sector and kernel of an operating system can be grafted together into a kernel image, which can be written to the initial sectors of the boot disk, such that the boot sector code is always at the head of the kernel image.

To create the kernel image, use the following command:

```bash
cat bootsect.bin kernel.bin > os-image
```

Now we need to write the boot sector that will bootstrap our kernel from a disk containing our kernel image (`os-image`).

### Boot Sector Code

```asm
[org 0x7c00]
KERNEL_OFFSET equ 0x1000 ; This is the memory offset to which we will load our kernel

mov [BOOT_DRIVE], dl ; BIOS stores our boot drive in DL, so it's best to remember this for later.
mov bp, 0x9000
mov sp, bp           ; Set up the stack.
mov bx, MSG_REAL_MODE
call print_string    ; Announce that we are starting booting from 16-bit real mode.
call load_kernel     ; Load our kernel
call switch_to_pm    ; Switch to protected mode, from which we will not return
jmp $

; Include our useful, hard-earned routines
%include "print/print_string.asm"
%include "disk/disk_load.asm"
%include "pm/gdt.asm"
%include "pm/print_string_pm.asm"
%include "pm/switch_to_pm.asm"

[bits 16]
; load_kernel
load_kernel:
    mov bx, MSG_LOAD_KERNEL
    call print_string
    mov bx, KERNEL_OFFSET
    mov dh, 15
    mov dl, [BOOT_DRIVE]
    call disk_load      ; Load the first 15 sectors (excluding the boot sector) from the boot disk
    ret

[bits 32]
; This is where we arrive after switching to and initializing protected mode.
BEGIN_PM:
    mov ebx, MSG_PROT_MODE
    call print_string_pm  ; Announce we are in protected mode.
    call KERNEL_OFFSET    ; Now jump to the address of our loaded kernel code
    jmp $                 ; Hang.

; Global variables
BOOT_DRIVE db
MSG_REAL_MODE db
MSG_PROT_MODE db
MSG_LOAD_KERNEL db

"Started in 16-bit Real Mode", 0
"Successfully landed in 32-bit Protected Mode", 0
"Loading kernel into memory.", 0

; Bootsector padding
times 510 - ($ - $$) db 0
dw 0xaa55
```

### Explanation of the Boot Sector Code:
1. **`[org 0x7c00]`**: This sets the starting address of the boot sector in memory (0x7c00).
2. **Kernel Loading**: The boot sector loads the kernel into memory starting at address `0x1000` using the `disk_load` function.
3. **Switch to Protected Mode**: The boot sector will then switch the CPU to 32-bit protected mode using the `switch_to_pm` routine.
4. **Print Messages**: Various strings are printed to indicate the status of the booting process, such as "Started in 16-bit Real Mode" and "Successfully landed in 32-bit Protected Mode."
5. **Boot Sector Padding**: The boot sector must be exactly 512 bytes, so padding is added to ensure this.

Now, you've created a basic kernel and boot sector that can load and execute the kernel in 32-bit protected mode.
