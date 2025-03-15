---
id: address-org
title: Boot Code Memory Offset
sidebar_label: Boot Code Memory Offset
---

Recall the memory layout in eXpOS. The zeroth page in memory was reserved for the ROM code and the bootstrap code was loaded from block 0 of the disk and placed in page 1 of the memory by the same code. While many other structures like the Process Table could have been placed anywhere in memory as we wished to, we were not able to change the location of the bootstrap code location from page 1. The same went for the ROM code in page 0. This is because the ROM code is the first interface between the machine and the operating system. Even if we were able to change the location of the ROM code, it would simply contradict the architectural specifications and prevent our operating system from booting as required. The x86 architecture loads block 0 (boot block) of the disk to the memory location 0x7c00 (the equivalent to address 512 in XSM memory). 

In the example from the previous section, we attempt to print the string "Entered 16 bit Real Mode" by passing its address to the print_string function through bx. The print_string function then accesses the memory content of bx and prints every character the same way.

It is clear that the print_string function expects the absolute address of the character in memory to be printed. `nasm` simply compiles the code such that only the offset of the label within the code is passed as the address of MSG_REAL_MODE. But since we use the print_string function after loading the boot sector into memory, the absolute address the MSG_REAL_MODE label is 0x7c00 (address where the boot sector is loaded) + the offset of the label in the boot sector code.

Adding 0x7c00 every time there is any memory reference within the boot sector code is messy and makes your code error-prone. Hence nasm provides us with a directive, `org`, which automatically offsets all such references by 0x7c00. Modify the `boot/boot.asm` to look like the one below.

<p class="codeblock-label">boot/boot.asm</p>

```
[org 0x7c00]

mov bx, MSG_REAL_MODE
call print_string
jmp $

%include "boot/print_string.asm"

MSG_REAL_MODE db "Entered 16 bit Real Mode", 0

times 510-($-$$) db 0
dw 0xaa55
```