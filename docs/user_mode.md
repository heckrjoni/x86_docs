---
id: user_mode
title: Transitioning to User Mode in x86 Architecture
sidebar_label: User Process
---


This document provides a comprehensive explanation and implementation of transitioning from kernel mode (ring 0) to user mode (ring 3) in the x86 architecture. It includes the setup of memory segments, Task State Segment (TSS), and privilege transitions.


## **1. Setting Up Segments for User Space**

User programs need to be isolated from each other to prevent interference, such as one program modifying another program's memory. Additionally, user programs must be restricted from modifying the kernel or accessing kernel-specific memory regions. This isolation is a fundamental requirement to maintain system integrity and security.

To achieve this, the memory segments provided to the user programs must differ from those allocated to the kernel. If user programs were allowed access to the same segments as the kernel, they could potentially access or modify kernel code and data, violating the security of the operating system. Therefore, separate segments are defined explicitly for user programs to enforce these restrictions.

**Read Prerequisite 1**


### ** Segment Descriptor Structure**
```c
struct seg_desc {
    uint32_t segment_limit1 : 16;  // Limit 0-15
    uint32_t base1 : 16;           // Base 0-15
    uint32_t base2 : 8;            // Base 16-23
    uint32_t type : 4;
    uint32_t S : 1;                // 0 for TSS, 1 for code/data
    uint32_t DPL : 2;              // Descriptor Privilege Level
    uint32_t P : 1;                // Present bit
    uint32_t segment_limit2 : 4;   // Limit 16-19
    uint32_t avl : 1;
    uint32_t O1 : 1;               // Reserved, set to 0
    uint32_t O2 : 1;               // 1 for 32-bit instructions
    uint32_t G : 1;                // Granularity bit
    uint32_t base3 : 8;            // Base 24-31
} __attribute__((packed));
```

### **Segment Selector Structure**
```c
struct segment_selector {
    uint16_t index : 13;  // Index in GDT
    uint16_t TI : 1;      // Table Indicator: 0 for GDT, 1 for LDT
    uint16_t RPL : 2;     // Requested Privilege Level
} __attribute__((packed));
```

### **Setting Segment Descriptors**
```c
void setUpSegDesc(struct seg_desc *a, uint32_t base, uint32_t limit, uint32_t DPL, uint32_t type, uint32_t s, uint32_t g, uint32_t o2) {
    a->base1 = base & 0xFFFF;
    a->base2 = (base >> 16) & 0xFF;
    a->base3 = (base >> 24) & 0xFF;
    a->segment_limit1 = limit & 0xFFFF;
    a->segment_limit2 = (limit >> 16) & 0xF;
    a->DPL = DPL & 0x3;
    a->type = type & 0xF;
    a->P = 1;             // Present bit
    a->S = s & 0x1;       // Code/data or TSS
    a->G = g & 0x1;       // Granularity
    a->O2 = o2 & 0x1;     // 32-bit instructions
    a->avl = 0;
    a->O1 = 0;
}
```

### **Example Usage**
```c
struct seg_desc tssdesc1, ldt_code_desc, ldt_data_desc;

// Set up user code and data segments
setUpSegDesc(&ldt_code_desc, 0x0, 0xFFFFF, 0x3, 0xA, 1, 1, 1); // Code segment
setUpSegDesc(&ldt_data_desc, 0x0, 0xFFFFF, 0x3, 0x2, 1, 1, 1); // Data segment
```


## **2. Task State Segment (TSS)**

The Task State Segment (TSS) is a special data structure in the x86 architecture used to store the context of a task. Originally, the TSS was integral to the hardware-based multitasking mechanism provided by x86 processors. It stored various pieces of information, including the CPU's register values, stack pointers, segment selectors, and more. Whenever a task switch occurred, the processor would automatically save the state of the current task in its TSS and load the state of the new task from its respective TSS. This mechanism allowed for efficient task switching entirely managed by the hardware, but it was eventually deemed inflexible and largely abandoned in favor of software-based task management.

In modern operating systems, the TSS is no longer used for its original purpose of hardware task switching. Instead, it has been repurposed to support specific functionalities, such as privilege-level transitions. For example, when transitioning from a lower privilege level (ring 3) to a higher privilege level (ring 0), the TSS provides a pointer to the kernel stack. This ensures that user-mode code cannot interfere with kernel-mode operations, as each privilege level has its own dedicated stack. The TSS is also used to handle double faults by providing a designated stack pointer for such critical situations, preventing cascading failures in the system.

Although the TSS's original role has diminished, its repurposed uses remain critical for modern operating system design. By utilizing the TSS to manage privilege transitions and ensure stack isolation, operating systems leverage its features to enhance security and reliability without relying on the outdated hardware task-switching model. This evolution highlights the adaptability of the x86 architecture and its ability to support software-driven innovations while maintaining backward compatibility.



## **3. Setting Up the TSS Segment**

### **TSS Structure**
```c
struct tss {
    uint16_t back_link, :16;
    uint32_t esp0;                   /**< Ring 0 stack virtual address. */
    uint16_t ss0, :16;               /**< Ring 0 stack segment selector. */
    uint32_t esp1;
    uint16_t ss1, :16;
    uint32_t esp2;
    uint16_t ss2, :16;
    uint32_t cr3;
    uint32_t eip;
    uint32_t eflags;
    uint32_t eax, ecx, edx, ebx;
    uint32_t esp, ebp, esi, edi;
    uint16_t es, :16;
    uint16_t cs, :16;
    uint16_t ss, :16;
    uint16_t ds, :16;
    uint16_t fs, :16;
    uint16_t gs, :16;
    uint16_t ldt, :16;
    uint16_t trace, bitmap;
} __attribute__((packed));
```

### **Example TSS Initialization**
```c
struct tss tss1;

setUpSegDesc(&tssdesc1, (uint32_t)&tss1, 0x68, 0x3, 0x9, 0x0, 0x0, 0x0);
```

---

## **4. Loading Descriptors into the GDT**

### **Loading GDT**
```c
void loadDescToGdt(struct seg_desc a) {
    uint64_t packed_desc = packSegDesc(&a);
    
    gdtEnd++;
    // Place the packed descriptor into the GDT at the current end
    *gdtEnd = packed_desc;

    return;
}

// Load descriptors
loadDescToGdt(ldt_code_desc);
loadDescToGdt(ldt_data_desc);
loadDescToGdt(tssdesc1);
```


## **5. Transitioning to User Mode**

---------Setting up stack for going into user program in assembly.--------
**Read Prerequisite 2**

Transitioning from kernel mode to user mode in the x86 architecture involves setting up the CPU state to operate at a lower privilege level (ring 3) while ensuring the environment is isolated and secure. This process is critical when starting a user program or returning to user mode after handling a system call. The transition requires setting up a user-mode stack, defining segment descriptors for user code and data, and using a mechanism like the IRET instruction to perform the privilege-level switch.

Before the transition, the kernel must allocate and configure resources for the user program. This includes defining separate GDT entries for user code and data segments, both with a Descriptor Privilege Level (DPL) of 3. Additionally, a user stack is allocated in memory, mapped appropriately in the page table, and marked as user-accessible. The kernel then prepares the state for the transition by pushing the user program's starting instruction pointer (EIP), the user code segment (CS), the initial stack pointer (ESP), and the user data segment (SS) onto the kernel stack. It also pushes the EFLAGS register with the interrupt flag enabled to allow the user program to receive interrupts.

The actual transition to user mode is performed using the IRET instruction. This instruction pops the values for EIP, CS, EFLAGS, ESP, and SS from the stack, setting the CPU state for user mode execution. Importantly, IRET automatically switches the CPU's Current Privilege Level (CPL) to the value specified in the lower two bits of the CS selector, which should be 3 for user mode. Once in user mode, the user program begins execution in its own isolated address space, with restricted access to kernel resources, enforced by the CPU's privilege and memory protection mechanisms. This process ensures a clean, secure transition while maintaining system stability.

------properties of stack during return, ireturn, interrupts----.
The stack plays a crucial role in managing control flow during operations like RET, IRET, and interrupts in the x86 architecture. During a function return (RET), the stack stores the return address pushed during the corresponding CALL instruction. When RET is executed, it pops the top value of the stack into the instruction pointer (EIP), transferring control back to the caller. This mechanism ensures seamless transitions between functions and maintains the integrity of nested calls. The stack's properties, such as its ability to grow downward and its Last In, First Out (LIFO) behavior, make it ideal for handling such operations.

For interrupts and privilege-level transitions, the stack takes on additional responsibilities. During an interrupt, the CPU automatically saves critical state information, including the instruction pointer (EIP), code segment (CS), and flags register (EFLAGS), onto the stack of the current privilege level. If the interrupt triggers a privilege-level change (e.g., from user mode to kernel mode), the CPU switches to a new stack designated by the Task State Segment (TSS) for the higher privilege level. The old stack pointer is preserved to allow returning to the previous context after the interrupt is handled. This automatic saving mechanism ensures that the interrupted program's state is fully recoverable.

When returning from an interrupt with the IRET instruction, the CPU restores the saved state from the stack. The IRET pops the EIP, CS, and EFLAGS values in sequence, resuming execution of the interrupted code as if the interrupt never occurred. If the interrupt involved a privilege-level transition, IRET also restores the stack pointer (ESP) and stack segment (SS) to their original values, returning to the correct privilege level. The stack’s ability to hold and restore these critical state components makes it a cornerstone of x86's interrupt handling and privilege isolation mechanisms, ensuring both stability and security during transitions.



**Refer to Prerequisite 3**


### **Assembly Code: Transitioning to User Mode**
```asm
changePrivLevel1:
    ; Prepare stack for IRET (returning to ring 3)
    push 0x23                 ; Ring 3 data segment selector for SS
    push 0xC000               ; Stack pointer for ring 3 for prog1
    pushf                     ; Push EFLAGS
    pop eax
    or eax, 0x200             ; Enable interrupts in EFLAGS
    push eax
    push 0x1B                 ; Ring 3 code segment selector for CS (index << 3) | RPL 3
    push userProg1            ; Entry point in ring 3

    ; Use IRET to switch to ring 3
    iret
```

### **User Program Example**
```asm
userProg1:
    mov ax, 0x23
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    call userProg11
    jmp $

void userProg11() {
    int a = 0;
    while (a < 10) {
        char *b = "0first\n";
        b[0] = '0' + a;
        printlib(b);
        a = a + 1;
    }
    return;
}
```





