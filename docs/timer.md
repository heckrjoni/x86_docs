---
id: timer
title: Setting up pcb and enabling timer interrupts
sidebar_label: Enabling timer and pcb
---

## 1. Setting Up the Timer Interrupt
- Configure the Programmable Interval Timer (PIT) to generate periodic interrupts.
- Write values to PIT registers to set the interrupt frequency.
- Implement `set_timer` function to initialize the timer.

```assembly
set_timer:
    mov al, 0x36      ; Set up PIT Channel 0 with square wave mode
    out 0x43, al      ; Write the command byte to PIT command register
    mov al, 0xFF      ; Load low byte of counter
    out 0x40, al      ; Write low byte to PIT Channel 0
    mov al, 0xFF      ; Load high byte of counter
    out 0x40, al      ; Write high byte to PIT Channel 0
    ret
```

## 2. Defining the Process Control Block (PCB)
- Create a structure to store process metadata, including process ID, state, and stack pointers.
- Initialize PCBs for two processes with different states.

```c
struct pcb_entry {
    int process_id;
    int state;
    uint32_t kernel_stack_entry;
    uint32_t kernel_stack_esp;
    uint32_t kernel_stack_ebp;
};

struct pcb_entry pcb1, pcb2;
int curr_process = 1;
```

## 3. Implementing the Timer Interrupt Handler
- Save register states using `pusha` and `popa`.
- Call the ISR function and send an End of Interrupt (EOI) command to the PIC.

```assembly
interrupt8:
    cli
    pusha
    push ds
    push es
    push fs
    push gs
    
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax

    call isrFn8
    
    pop gs
    pop fs
    pop es
    pop ds
    popa
    
    mov al, 0x20
    out 0x20, al
    sti
    iret
```

## 4. Implementing Context Switching
- Save the current stack pointer (`ESP` and `EBP`).
- Switch between processes, updating their states and stack pointers.
- Restore the new process's stack before execution resumes.

```c
void contextswitch() {
    uint32_t esp_temp, ebp_temp;
    asm volatile("mov %%esp, %0" : "=r" (esp_temp));
    asm volatile("mov %%ebp, %0" : "=r" (ebp_temp));

    if (tick == 0) {
        tick++;
        return;
    }
    
    if (curr_process == 1) {
        pcb1.state = READY;
        pcb1.kernel_stack_esp = esp_temp;
        pcb1.kernel_stack_ebp = ebp_temp;
        tss1.esp0 = pcb2.kernel_stack_entry;
        curr_process = 2;

        if (pcb2.state == CREATED) {
            pcb2.state = RUNNING;
            changePrivLevel2();
        } else {
            pcb2.state = RUNNING;
            esp_temp = pcb2.kernel_stack_esp;
            ebp_temp = pcb2.kernel_stack_ebp;
        }
    } else {
        pcb2.state = READY;
        pcb1.state = RUNNING;
        pcb2.kernel_stack_esp = esp_temp;
        pcb2.kernel_stack_ebp = ebp_temp;
        tss1.esp0 = pcb1.kernel_stack_entry;
        curr_process = 1;
        esp_temp = pcb1.kernel_stack_esp;
        ebp_temp = pcb1.kernel_stack_ebp;
    }
    
    asm volatile("mov %0, %%esp" :: "r" (esp_temp));
    asm volatile("mov %0, %%ebp" :: "r" (ebp_temp));
}
```

## 5. Privilege Level Switching
- Call `changePrivLevel2()` to transition the current process to a different privilege level.

```c
extern void changePrivLevel2();
```

## 6. Debugging and Logging
- Convert and print stack pointer values for debugging using `uint32_to_hex()`.
- Use `printker()` to log context switch events.

```c
void uint32_to_hex(uint32_t num, char hex_str[]) {
    char hex_digits[] = "0123456789ABCDEF";
    hex_str[0] = '0';
    hex_str[1] = 'x';

    for (int i = 0; i < 8; i++) {
        int shift = (7 - i) * 4;
        hex_str[i + 2] = hex_digits[(num >> shift) & 0xF];
    }
    hex_str[10] = '\0';
}
```

## 7. Setting Up the PCB
- Initialize process control blocks before execution starts.

```c
void setup_pcb() {
    pcb1.process_id = 1;
    pcb1.state = RUNNING;
    pcb1.kernel_stack_entry = 0xd000;

    pcb2.process_id = 2;
    pcb2.state = CREATED;
    pcb2.kernel_stack_entry = 0xd700;
}
```

---

