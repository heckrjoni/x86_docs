---
id: Introduction to the GDT
title: Understanding the Global Descriptor Table (GDT) and Segmentation
sidebar_label: Introduction to the GDT
---


It is important to understand the main point of this GDT, that is so fundamental to the operation of protected mode.  
GDT is a data structure for implementing segmentation in an OS.

Before delving into the details of a GDT setup, have a quick read about segmentation.  
Segmentation and paging are two important memory management techniques employed by an OS.  
Memory segmentation is a memory management technique in operating systems that divides memory into variable-sized segments.  
For this, the memory is divided (logically) into segments for each of code, data, stack...

The memory address is accessed with the help of two registers, segment base and segment offset.

Each segment is identified by a segment selector (or segment register in x86).  
It is calculated by combining the segment's base address and the offset:  
Translated into a physical address using the equation:  

Physical address = (Segment Selector * 0x10) + offset

This multiplication by 0x10 is specific to the x86 architecture and reflects a left shift of 4 bits.  

Segments in x86 can overlap. Since segments are defined by their base address and size, two segments may share part of their address ranges. For example:  
- Segment `0x1000` starts at physical address `0x10000` and extends to `0x1FFFF`.  
- Segment `0x1010` starts at physical address `0x10100` and extends to `0x200FF`.  

In this case, the range `0x10100` to `0x1FFFF` is accessible through both segments.

The Global Descriptor Table (GDT) is a data structure that holds segment descriptors, which define the properties of memory segments in a system. These segment descriptors specify how a segment is accessed, its size, location, and permissions.  
Each entry in the GDT describes one segment in the system.

## Segment Descriptor

A segment descriptor is an 8-byte entry in the GDT that contains detailed information about a segment. It defines:  
- **Base address (32 bits):** Defines where the segment begins in physical memory.  
- **Segment Limit (20 bits):** Defines the size of the segment.  
- **Various flags:** Affect how the CPU interprets the segment, such as the privilege level of code that runs within it or whether it is read- or write-only.

## Accessing the GDT and Segment Descriptors

Programs use segment selectors to refer to a segment.  
A selector is a 16-bit value that contains:  
- An index into the GDT.  
- The Requested Privilege Level (RPL).  
- A Table Indicator (TI) to select GDT or LDT (Local Descriptor Table) (we'll cover later).

### Translation Process

The CPU uses the segment selector to locate the corresponding descriptor in the GDT.  
The descriptor provides the base address, limit, and access rights, allowing the CPU to compute the physical address and verify permissions.

The GDT enables segmentation-based memory protection, ensuring that:  
- Code cannot accidentally or maliciously write into data or system-critical segments.  
- The GDT is pointed to by a special CPU register called the **GDTR** (Global Descriptor Table Register).  
- A segment can only be accessed if the privilege level of the process matches the segment’s Descriptor Privilege Level (DPL).  

If a user program tries to access a restricted segment, the CPU raises a **General Protection Fault**.  
When the CPU accesses a memory segment, it performs checks using the GDT.

As we only need to know the basics for now, we'll leave the remaining of GDT and segmentation to the students.  

## Additional Resources

- [Segmentation - OSDev Wiki](https://wiki.osdev.org/Segmentation)  
- [Global Descriptor Table - OSDev Wiki](https://wiki.osdev.org/Global_Descriptor_Table)

