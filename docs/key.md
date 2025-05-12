---
id: keyboard
title: Implementing Keyboard interrupt
sidebar_label: Keyboard interrupt
---
# Keyboard Input Handling in OS Development

## Overview
Keyboard input is managed through a combination of hardware interrupts, scancode processing, and buffer management. When a key is pressed, the keyboard controller generates a scancode, which is transmitted to the CPU via the Programmable Interrupt Controller (PIC). The operating system must then process this scancode, map it to the corresponding ASCII character, and store it in a buffer for further processing.

## 1) Low-Level I/O Port Communication Functions
This code provides two essential functions, `inb()` and `outb()`, which enable communication with hardware devices through I/O ports. These functions are critical for interacting with system components such as the keyboard, PIC (Programmable Interrupt Controller), and other peripherals. The `inb()` function reads a single byte (8 bits) of data from the specified I/O port. It uses the `inb` assembly instruction to retrieve the value from the given port and stores it in the result variable. The `outb()` function sends a byte of data to the specified I/O port. It uses the `outb` assembly instruction. These functions are fundamental for direct hardware control in low-level OS development, enabling the system to send and receive data from devices like the keyboard controller, interrupt controllers, and other I/O devices.

### `io.c`
```c
#include "io.h"

uint8_t inb(uint16_t port) {
    uint8_t result;
    asm volatile ("inb %1, %0" : "=a"(result) : "Nd"(port));
    return result;
}

void outb(uint16_t port, uint8_t data) {
    asm volatile ("outb %0, %1" : : "a"(data), "Nd"(port));
}
```

## 2) Understanding the Programmable Interrupt Controller (PIC) and Remapping
The Programmable Interrupt Controller (PIC) is a hardware component that helps the CPU manage interrupts—signals from devices like the keyboard, mouse, and system timer that require the CPU's attention. Each interrupt is assigned a unique interrupt vector, which tells the CPU where to find the code to handle the interrupt. However, by default, the PIC assigns interrupt vectors that conflict with critical CPU exceptions. For example, the Master PIC originally assigns interrupt numbers 8–15, while the Slave PIC assigns 112–119. This creates a problem because some of these numbers are already used by the CPU for error handling, such as interrupt 8 for the double fault exception.

To prevent conflicts, we remap the PIC so that the Master PIC uses interrupts 32–39 and the Slave PIC uses interrupts 40–47. This ensures that hardware interrupts do not overlap with CPU exception handlers. The remapping process involves sending specific commands to the PIC ports (0x20 and 0xA0 for commands, 0x21 and 0xA1 for data). First, we initialize both PICs by sending a command word that tells them to enter initialization mode and expect further configuration. Then, we specify new interrupt vector bases so that the Master PIC starts at 32 and the Slave PIC starts at 40 (0x28 in hex).

Since the Slave PIC is connected to the Master PIC via IRQ2, we also need to configure how they communicate using ICW3. The Master PIC is informed that the Slave is at IRQ2, and the Slave PIC acknowledges that it is connected to the Master. Finally, we set both PICs to 8086 mode (ICW4), ensuring compatibility with modern CPUs. After configuration, we clear the interrupt mask, allowing all IRQs to be processed again.

### PIC Remapping Code
```c
void remap_pic() {
    outb(0x20, 0x11);
    outb(0xA0, 0x11);
    outb(0x21, 0x20);
    outb(0xA1, 0x28);
    outb(0x21, 0x04);
    outb(0xA1, 0x02);
    outb(0x21, 0x01);
    outb(0xA1, 0x01);
    outb(0x21, 0x0);
    outb(0xA1, 0x0);
}
```

## 3) Enabling IRQ1 (Keyboard Interrupt)
When we remap the PIC, we initially unmask all IRQs by writing 0x00 to the PIC's mask registers (0x21 for the Master PIC and 0xA1 for the Slave PIC). However, in a real operating system, we usually re-mask (disable) all IRQs after remapping to prevent unnecessary interrupts before proper handlers are set up. This means that, by default, after remapping, all IRQs remain disabled until we explicitly enable (unmask) them one by one.

### Enabling IRQ1
```c
void enable_irq1() {
    outb(0x21, 0xFD);
    outb(0xA1, 0xFF);
}
```

The keyboard generates interrupts on IRQ1, which is controlled by bit 1 in the Master PIC’s interrupt mask register. If this bit is set to 1, IRQ1 is disabled. If this bit is set to 0, IRQ1 is enabled.

## 4) Setting Up the IDT
Mapping the keyboard interrupt (IRQ1) to vector 33 (0x21).

```c
setUpIntDesc(33, (uintptr_t)interrupt33, 0x08, 0x8E);
```

## 5) Interrupt Handler
The CPU reads the scancode from the keyboard data port and processes it.

### ISR Function
```c
void isrFn33() {
    uint8_t scancode = inb(0x60);
    process_scancode(scancode);
    send_eoi(1);
}
```

### End of Interrupt (EOI)
```c
void send_eoi(uint8_t irq) {
    if (irq >= 8) outb(0xA0, 0x20);
    outb(0x20, 0x20);
}
```

## 6) Processing Scancodes and Managing the Terminal Buffer
This code handles keyboard input by reading scancodes, converting them into ASCII characters, and storing them in a buffer for processing.

### Processing Scancode
```c
void process_scancode(uint8_t scancode) {
    if (scancode & 0x80) return;
    char ascii = scancode_to_ascii[scancode];
    if (ascii) {
        if (ascii == '\n') {
            terminal_buffer[buffer_index] = '\0';
            input_ready = 1;
        } else if (buffer_index < BUFFER_SIZE - 1) {
            terminal_buffer[buffer_index++] = ascii;
        }
    }
}
```

### Reading Terminal Input
```c
char *read_terminal() {
    while (!input_ready);
    input_ready = 0;
    terminal_buffer[buffer_index] = '\0';
    buffer_index = 0;
    return terminal_buffer;
}
```

## Conclusion
This project successfully implements keyboard input handling in an OS environment, including direct hardware communication, interrupt handling, and buffer management. Future improvements include handling special keys and implementing enhanced scancode mapping.

