---
id: io-intro
title: Introduction to I/O Addressing
sidebar_label: Introduction to I/O Addressing
---

This section outlines the different ways to to read from or write to Input/Output devices. Recall the different addressing modes discussed in your Operating Systems theory course. We shall discuss two modes of addressing here that is supported by the x86 architecture.

1. Port-mapped I/O (also known as I/O-mapped I/O)
2. Memory-mapped I/O

### Port-mapped I/O

In this mode of addressing, I/O devices are addressed using a separate I/O address space. This is different from the physical RAM address space. The addresses are transferred via the address bus and the data is transferred to and from the devices. I/O devices are connected to different ports on the machine and these port are mapped to the I/O address space. This form of addressing was very common in older systems where there was little physical memory to map I/O addresses too.

### Memory-mapped I/O

In this mode of addressing, I/O devices being addressed share the same address space as the physical memory. This enables the same opcodes/commands used for moving data in registers and memory to be used for I/O devices as well. A certain portion of the physical memory is reserved for some I/O devices in this addressing. Moving data to or from those locations writes or reads from the attached I/O device for that location.

In x86, the screen in protected mode can be accessed via 