---
id: segmentation
title: Memory Organisation and Setting up Stack
sidebar_label: Memory Organisation and Setting up Stack
---

This chapter is very vaguely written. The aim is simply to introduce the idea of memory segmentation. While one will definitely develop better clarity on the idea in further stages, if you feel you must absolutely understand the concept better before you can proceed, you can refer the Intel 80386 Programmer's Reference Manual Chapter 5. It is, however, advised to not refer it right now as it is not required presently.

The x86 allows the system-software designers to choose different models of memory segmentation. The different models were discussed in your Operating Systems theory course. There are two memory organisation schemes discussed in the Intel 80386 Programmer's Reference Manual. These are the flat model and the segmented model.

In the flat model, the logical memory is seen as a flat space by the application developers i.e. a single stretch of linear addresses. As the addressing in 80386 is 32-bit, A maximum of 2<sup>32</sup> bytes RAM can be addressed (4 GB). Note that it is not necessary that this logical address space from 0 to 2<sup>32</sup>-1 need not actually be contiguous in the physical address space. The logical to physical address mapping is upto the system-software designer and application designers need not know about them.

A second organisation model and the one we will be using for our project is the segmented model. In this model, the memory as seen by the application programmers as segments of upto 4 GB each. There can be upto 16.383 segments of linear addresses. Again, all address mapping to the actual physical address space is kept unknown to the application developers.

The above descriptions on the models are very brief and it is possible to not be yet solid on its working. However, we shall not go into detail of memory organisation models and the advantage of one model over the other. It is not necessary for you to know the same either for now. All that is required to understand is how x86 offers support for a segmented memory model where you can place your application's code and instructions in one segment, data in another segment, stack in another and so on. There are various registers such as CS (Code Segment Register), DS (Data Segment Register), SS (Stack Segment Register), ES (Extra Segment Register) etc. that enable this. The x86 automatically translates the addresses in an instruction with the correct segment according to the address type. For example, all instructions (IP) will be fetched from the Code Segment while all references to the register SP (Stack Pointer) will use the Stack Segment and all memory data references will use the Data Segment. The CS, DS, SS, ES are all 16-bit registers. ith a segmented model, all corresponding address references in the code is simply the offset in the respective segment. In addition the actual translation is not done by simply appending the offset address to the segment address but by merging them as `(segment_address << 4) + offset`. The idea will become clearer with further stages.

When the CPU is RESET (powered on), all the segment registers are initialised to zero (0x0). We shall now initialize a stack by setting the value of the register `sp`. Simply add lines `mov bp, 0x9000` and `mov sp, bp` to initialize our real mode stack frame and top of stack to the address `0x9000`. The actual address for stack operations now will be (0x0 << 4 + 0x9000 = 0x9000).

<p class="codeblock-label">boot/boot.asm</p>

```
[org 0x7c00]

mov bx, MSG_REAL_MODE
call print_string

mov bp, 0x9000
mov sp, bp

jmp $

%include "boot/print_string.asm"

MSG_REAL_MODE db "Entered 16 bit Real Mode", 0

times 510-($-$$) db 0
dw 0xaa55
```

Recall that a stack grows downwards. Hence in our case, upon using the push command once, the stack top is updated to 0x8ffe (2 bytes downwards). The reason for the two bytes is that we have a 16-bit stack and not an 8-bit one and hence all push and pop instruction deals with 2 bytes (16-bits) at once. You can test the same as follows:

<p class="codeblock-label">boot/example.asm</p>

```
mov bp, 0x9000
mov sp, bp

push 'A'
mov ah, 0x0e
mov al, [0x8ffe] ; -2 due to 16-bit stack
int 0x10

jmp $

times 510-($-$$) db 0
dw 0xaa55
```

Run the above code by compiling with `nasm -f bin -o example boot/example.asm` and running with QEMU `qemu-system-i386 -hda example`
We will also look at an example for understanding segmentation. We will attempt to store two characters in the same offset but in two different segments one starting at 0x0 and the other at 0x1000 and try to access them.

<p class="codeblock-label">boot/example.asm</p>

```
mov bp, 0x9000
mov sp, bp
push 'A' ; Stored at (0x0 + 0x8ffe)

mov bx, 0x100 ; Segment addresses are shifted automatically << 4 (i.e. 0x1000)
mov ss, bx ; Since x86 does not allow storing immediate values 
           ; directly to segment registers, we use bx here
mov sp, 0x9000 ; Resetting stack top to 0x9000
push 'B' ; Stored at (0x1000 + 0x8ffe)

mov ah, 0x0e
mov al, [0x8ffe] ; A memory reference implicitly uses data segment value
                 ; and DS initialises to 0x0
int 0x10

mov al, [ss:0x8ffe] ; Explicitly using a different segment for the address
int 0x10

jmp $

character: db 'B'
times 510-($-$$) db 0
dw 0xaa55
```

After understanding the above examples, remove the example.asm file from the `boot/` directory.

Now that we have set up our stack, let us modify the print_string function to save the value of ax before operating with it and restore it before returning

<p class="codeblock-label">boot/print_string.asm</p>

```
print_string:
	push ax ; Save ax register
	mov ah, 0x0e ; Specify the function to use i.e. print al to tty
	print:
		mov al, [bx]
		int 0x10
		inc bx
		cmp al, 0
		je return
		jmp print
	return:
	pop ax ; Restore ax register
	ret
```