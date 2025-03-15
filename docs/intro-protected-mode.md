---
id: intro-protected-mode
title: Introduction to the Protected Mode
sidebar_label: Introduction to the Protected Mode
---

It would be nice to continue working in the 16-bit real mode with which we have now become much better acquainted, but in order to make fuller use of the CPU, and to better understand how developments of CPU architectures can benefit modern operating systems, namely memory protection in hardware, we must press on into 32-bit protected mode.

## The main differences in 32-bit protected mode are:
- **Registers are extended to 32 bits**, with their full capacity being accessed by prefixing an `e` to the register name, for example: `mov ebx, 0x274fe8fe`.
- For convenience, there are **two additional general-purpose segment registers**, `fs` and `gs`.
- **32-bit memory offsets** are available, so an offset can reference a whopping 4 GB of memory (0xffffffff).
- The CPU supports a more sophisticated — though slightly more complex — means of **memory segmentation**, which offers two big advantages:
  - Code in one segment can be prohibited from executing code in a more privileged segment, so you can protect your kernel code from user applications.
  - The CPU can implement **virtual memory** for user processes, such that pages (i.e., fixed-sized chunks) of a process’s memory can be swapped transparently between the disk and memory on an as-needed basis. This ensures main memory is used efficiently, in that code or data that is rarely executed needn’t hog valuable memory.
- **Interrupt handling** is also more sophisticated.

The most difficult part about switching the CPU from 16-bit real mode into 32-bit protected mode is that we must prepare a complex data structure in memory called the **Global Descriptor Table (GDT)**, which defines memory segments and their protected-mode attributes. We have to define the GDT in assembly language.

We can no longer use **BIOS** once switched into 32-bit protected mode. BIOS routines, having been coded to work only in 16-bit real mode, are no longer valid in 32-bit protected mode.

For more information, check the article on [OSDev Wiki - Protected Mode](https://wiki.osdev.org/Protected_Mode).

