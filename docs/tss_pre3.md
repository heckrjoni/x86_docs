---
id: tss_preq3
title: Segment Descriptors in x86 (32-bit)
sidebar_label: Prerequisite 3
---


A segment descriptor defines the attributes of a memory segment in the x86 architecture. It resides in the Global Descriptor Table (GDT) or Local Descriptor Table (LDT) and provides essential details for segmentation, including the segment's size, base address, and access permissions.
Structure of a Segment Descriptor:

A segment descriptor is 8 bytes (64 bits) and has the following fields:

    Base Address (32 bits total):
        Specifies the starting address of the segment.
        Divided across multiple fields: Base Low (16 bits), Base Middle (8 bits), and Base High (8 bits).

    Segment Limit (20 bits):
        Defines the size of the segment.
        Split into Limit Low (16 bits) and Limit High (4 bits).

    Type and Access Flags:
        Describes the segment type and access permissions:
            Code/Data: Identifies whether the segment contains code, data, or system information.
            Executable: Indicates if the segment contains executable code.
            Writable/Readable: Specifies read or write permissions.
            Accessed: Set by the CPU when the segment is accessed.

    Descriptor Privilege Level (DPL):
        Defines the privilege level (0 to 3) required to access the segment.

    Granularity and Flags:
        Granularity (G): Controls the segment size interpretation:
            0: Limit in bytes.
            1: Limit in 4 KB units (pages).
        Default Operation Size (D): Defines the default instruction size (16-bit or 32-bit).

    Base Address (32 bits total):
        Specifies the starting address of the segment.
        Divided across multiple fields: Base Low (16 bits), Base Middle (8 bits), and Base High (8 bits).

    Segment Limit (20 bits):
        Defines the size of the segment.
        Split into Limit Low (16 bits) and Limit High (4 bits).

    Type and Access Flags:
        Describes the segment type and access permissions:
            Code/Data: Identifies whether the segment contains code, data, or system information.
            Executable: Indicates if the segment contains executable code.
            Writable/Readable: Specifies read or write permissions.
            Accessed: Set by the CPU when the segment is accessed.

    Descriptor Privilege Level (DPL):
        Defines the privilege level (0 to 3) required to access the segment.

    Granularity and Flags:
        Granularity (G): Controls the segment size interpretation:
            0: Limit in bytes.
            1: Limit in 4 KB units (pages).
        Default Operation Size (D): Defines the default instruction size (16-bit or 32-bit).

Example Use Cases:

    Kernel Segments: Have DPL=0, allowing access only by privileged code.
    User Segments: Have DPL=3, restricting access to user-mode code.

Segment descriptors are integral to the x86 segmentation mechanism, enabling fine-grained control over memory access and providing the foundation for memory protection.
