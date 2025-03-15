---
id: tss_preq2
title: Memory Protection Mechanisms
sidebar_label: Prerequisite 2
---
Memory Protection Mechanisms

Several methods are used in modern operating systems to ensure memory protection:
1. Segmentation

    In the x86 architecture, segmentation divides memory into different logical segments, each defined by a descriptor in the Global Descriptor Table (GDT) or Local Descriptor Table (LDT).
    By setting different privilege levels (DPL), access to certain segments can be restricted:
        Kernel segments are assigned the highest privilege level (CPL=0).
        User segments are assigned the lowest privilege level (CPL=3).
    Segment boundaries and privilege levels ensure that user programs cannot access kernel segments.

2. Paging

    Paging further enhances memory protection by dividing the address space into fixed-size pages.
    The page table maps virtual memory to physical memory and enforces access permissions for each page:
        Read, write, and execute permissions can be set for each page.
        Kernel pages can be marked as accessible only in supervisor mode (ring 0).
        User pages are accessible only in user mode (ring 3).
    This mechanism ensures that even if a user program tries to access restricted memory, the Memory Management Unit (MMU) triggers a page fault, which the kernel can handle.

3. Privilege Levels

    The x86 architecture uses four privilege levels (rings) to control access to resources:
        Ring 0: Kernel code and data.
        Ring 3: User programs.
    Transitions between privilege levels occur through well-defined mechanisms, such as system calls, ensuring controlled interaction between user programs and the kernel.   
