---
id: Defining GDT in Assembly
title: Defining GDT in Assembly
sidebar_label: Defining GDT in Assembly
---

# Defining GDT in Assembly

We have already seen examples of how to define data within our assembly code, using the `db`, `dw`, and `dd` assembly directives, and these are exactly what we must use to put in place the appropriate bytes in the segment descriptor entries of our GDT.

We don’t actually directly give the CPU the start address of our GDT but instead give it the address of a much simpler structure called the GDT descriptor.

The GDT is a 6-byte structure containing:
- **GDT size** (16 bits)
- **GDT address** (32 bits)

### GDT CODE

```asm
; GDT
gdt_start :
gdt_null : ; the mandatory null descriptor
    dd 0x0 ; ’ dd ’ means define double word ( i.e. 4 bytes )
    dd 0x0

gdt_code : ; the code segment descriptor
    ; base = 0x0 , limit = 0xfffff ,
    ; 1st flags : ( present )1 ( privilege )00 ( descriptor type )1 -> 1001b
    ; type flags : ( code )1 ( conforming )0 ( readable )1 ( accessed )0 -> 1010b
    ; 2nd flags : ( granularity )1 (32-bit default )1 (64-bit seg )0 ( AVL )0 -> 1100b
    dw 0xffff
    ; Limit ( bits 0-15)
    dw 0x0
    ; Base ( bits 0-15)
    db 0x0
    ; Base ( bits 16-23)
    db 10011010b ; 1st flags , type flags
    db 11001111b ; 2nd flags , Limit ( bits 16-19)
    db 0x0
    ; Base ( bits 24-31)

gdt_data : ; the data segment descriptor
    ; Same as code segment except for the type flags :
    ; type flags : ( code )0 ( expand down )0 ( writable )1 ( accessed )0 -> 0010b
    dw 0xffff
    ; Limit ( bits 0-15)
    dw 0x0
    ; Base ( bits 0-15)
    db 0x0
    ; Base ( bits 16-23)
    db 10010010b ; 1st flags , type flags
    db 11001111b ; 2nd flags , Limit ( bits 16-19)
    db 0x0
    ; Base ( bits 24-31)

gdt_end :

    ; The reason for putting a label at the end of the
    ; GDT is so we can have the assembler calculate
    ; the size of the GDT for the GDT descriptor (below)

; GDT descriptor
gdt_descriptor :
    dw gdt_end - gdt_start - 1
    dd gdt_start
    ; Size of our GDT, always less one
    ; of the true size
    ; Start address of our GDT

; Define some handy constants for the GDT segment descriptor offsets,
; which are what segment registers must contain when in protected mode. 
; For example, when we set DS = 0x10 in PM, the CPU knows that we mean it to use the
; segment described at offset 0x10 (i.e., 16 bytes) in our GDT, which in our
; case is the DATA segment (0x0 -> NULL; 0x08 -> CODE; 0x10 -> DATA)

CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start
```

## Making the Switch

The first thing we have to do is disable interrupts using the `cli` (clear interrupt) instruction, which means the CPU will simply ignore any future interrupts that may happen, at least until interrupts are later enabled.

This is very important, because, like segment-based addressing, interrupt handling is implemented completely differently in protected mode than in real mode. The current IVT (Interrupt Vector Table) set up by the BIOS at the start of memory is completely meaningless. Even if the CPU could map interrupt signals to their correct BIOS routines (e.g., when the user presses a key, storing its value in a buffer), the BIOS routines would execute 16-bit code, which will have no concept of the 32-bit segments we defined in our GDT and will ultimately crash the CPU by having segment register values that assume the 16-bit real mode segmenting scheme.

The next step is to tell the CPU about the GDT that we just prepared. We use a single instruction to do this, passing the GDT descriptor:
```asm
lgdt [ gdt_descriptor ]
```

Then, we make the actual switch over by setting the first bit of a special CPU control register, `cr0`. We cannot set that bit directly in the register, so we must load it into a general-purpose register, set the bit, and then store it back into `cr0`.

```asm
mov eax, cr0
or eax, 0x1
mov cr0, eax
```

To make the switch to protected mode, we set the first bit of CR0, a control register. We also need to execute a jump to flush the CPU pipeline and process new 32-bit instructions immediately after instructing the CPU to switch mode. A far jump will force the CPU to flush the pipeline, which completes all instructions currently in different stages of the pipeline.

A far jump, as opposed to a near (standard) jump, provides the target segment as follows:
```asm
jmp <segment>:<address offset>
```

### Reusable Routine

We can combine the entire process into a reusable routine:

```asm
[bits 16]
; Switch to protected mode
switch_to_pm:
    cli
    ; Disable interrupts until we have set up the protected mode interrupt vector
    lgdt [gdt_descriptor] ; Load our global descriptor table
    mov eax, cr0
    or eax, 0x1
    mov cr0, eax ; Set the first bit of CR0 to switch to protected mode
    jmp CODE_SEG: init_pm ; Far jump to code to flush the pipeline

[bits 32]
; Initialize registers and the stack once in PM
init_pm:
    mov ax, DATA_SEG
    mov ds, ax
    mov ss, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    ; Now in PM, our old segments are meaningless,
    ; so we point our segment registers to the data selector defined in our GDT
    mov ebp, 0x90000
    mov esp, ebp ; Update stack pointer to top of free space
    call BEGIN_PM ; Finally, call some well-known label
```

---



