---
id: multi
title: MultiProgramming x86 (32-bit)
sidebar_label: Switching between 2 user processes
---



# User Stack and Kernel Stack in Multiprogramming

In a multiprogramming environment, the operating system (OS) must manage multiple processes efficiently, ensuring isolation, resource sharing, and proper execution. Two key concepts in this regard are the **user stack** and the **kernel stack**:

## User Stack
- Belongs to the **user space**.
- Stores **function arguments, local variables, and return addresses** for functions invoked during user-mode execution.
- Each process has its own **user stack** for user-mode operations, which resides in its private virtual address space.

## Kernel Stack
- Belongs to the **kernel space**.
- Stores the **state of the process when it executes kernel code** (e.g., during system calls, interrupts, or exceptions).
- Each process has its own **kernel stack** (separate from the user stack) to handle operations in kernel mode.
- Typically, the kernel stack is **smaller** than the user stack and is allocated in kernel memory, often as a fixed-size structure.

## Switching Between User and Kernel Stacks
When a process transitions between **user mode** and **kernel mode** (e.g., during a system call or interrupt), the CPU switches from the **user stack** to the **kernel stack**:

1. The CPU saves the **current state** (program counter, registers, etc.) on the kernel stack.
2. Execution begins at the corresponding **kernel entry point**.
3. When the kernel work is complete, the CPU restores the **saved state** from the kernel stack and returns to user mode.

---

# Context Switching in x86 (32-bit) Multiprogramming

## Context Switching
**Context switching** is the process of saving and restoring the CPU state so that execution can move from one process (or thread) to another. This is a critical component of multiprogramming and is managed by the operating system.

### Components of Context in x86 (32-bit)
1. **Registers**:
   - General-purpose registers: `EAX`, `EBX`, `ECX`, `EDX`, `ESI`, `EDI`, `EBP`, `ESP`.
   - Segment registers: `CS`, `DS`, `ES`, `FS`, `GS`, `SS`.
   - Instruction Pointer (`EIP`).
   - Flags Register (`EFLAGS`).

2. **Memory State**:
   - The **user and kernel stack pointers** (`ESP` and `SS`).
   - **Virtual memory mappings** (page tables).

3. **Process Control Block (PCB)**:
   - Stores the **state of a process**, including its context, priority, and other metadata.

---

## Interrupts Performing Context Switching

### Assembly Code for Interrupts
```asm
interrupt1:
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    mov [0xb0000], esp
    call isrFn1
    call changePrivLevel2

interrupt2:
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    mov edi, esp
    call isrFn2
    mov esp, [0xb0000]
    mov ax, 0x23
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    iret
    
interrupt3:
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    mov [0xb0000], esp
    call isrFn3
    
    mov esp, [0xb0100]
    mov ax, 0x23
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    
    iret
```

---

## Key Features in x86 for Context Switching

### 1. Task State Segment (TSS)
The **TSS** is a data structure used to store **task-specific information**, including the kernel stack pointer for each process.

### 2. Privilege Levels
- x86 uses four **privilege levels** (ring 0 to ring 3).
- Context switching often involves transitions between **ring 3** (user mode) and **ring 0** (kernel mode).

### 3. Paging
- The **Memory Management Unit (MMU)** supports virtual memory, allowing processes to have **isolated memory spaces**.
- The page directory base register (`CR3`) is switched during context switching to load a new process's page tables.

---

## Functions for Changing the Kernel Stack
```c
void isrFn1() {
    tss1.esp0 = 0xd700;
    return;
}

void isrFn2() {
    tss1.esp0 = 0xd000;
    return;
}

void isrFn3() {
    tss1.esp0 = 0xd700;
    return;
}
```

---

# User Programs

### `userProg11`
```c
void userProg11() {
    int a = 0;
    while (a < 10) {
        char *b = "0first\n";
        b[0] = '0' + a;
        printlib(b);
        conSwitch(1);
        a = a + 1;
    }
    return;
}
```

### `userProg22`
```c
void userProg22() {
    int a = 0;
    while (a < 10) {
        char *b = "0scnd\n";
        b[0] = '0' + a;
        printlib(b);
        conSwitch(2);
        a = a + 1;
    }
    return;
}
```

---

## Software Context Switch Function
```c
void conSwitch(int a) {
    if (a == 1 && p2 == 0) {
        p2 = 1;
        asm volatile ("int $0x01");
    } else if (a == 1 && p2 == 1) {
        asm volatile ("int $0x03");
    } else if (a == 2) {
        asm volatile ("int $0x02");
    }
}
```

---

# Initializing the Two User Programs

### `changePrivLevel1`
```asm
changePrivLevel1:
    push 0x23               ; Ring 3 data segment selector for SS
    push 0xC000             ; Stack pointer for ring 3 FOR prog1
    pushf                   ; Push EFLAGS
    pop eax
    or eax, 0x200           ; Enable interrupts in EFLAGS
    push eax
    push 0x1B               ; Ring 3 code segment selector for CS
    push userProg1          ; Entry point in ring 3
    iret                    ; Use IRET to switch to ring 3
```

### `changePrivLevel2`
```asm
changePrivLevel2:
    push 0x23               ; Ring 3 data segment selector for SS
    push 0xC700             ; Stack pointer for ring 3 FOR prog2
    pushf                   ; Push EFLAGS
    pop eax
    or eax, 0x200           ; Enable interrupts in EFLAGS
    push eax
    push 0x1B               ; Ring 3 code segment selector for CS
    push userProg2          ; Entry point in ring 3
    iret                    ; Use IRET to switch to ring 3
```

### User Programs Entry Points
```asm
userProg1:
    mov ax, 0x23
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    call userProg11
    jmp $

userProg2:
    mov ax, 0x23
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    call userProg22
    jmp $
```
```
