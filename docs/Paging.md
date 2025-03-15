---
id: Paging
title: Paging in 32-Bit Systems
sidebar_label: Implementing Paging
---



In 32-bit paging, the memory management system uses two levels of tables: a **Page Directory** and **Page Tables**. 

## Overview

- The **Page Directory** consists of 1024 entries, each pointing to a **Page Table**.
- Each **Page Table** also contains 1024 entries, mapping specific virtual memory pages to their corresponding physical memory addresses.
- Both structures must be page-aligned (address must be a multiple of 4096 bytes) for compatibility with the CPU’s memory management unit (MMU).

### Address Translation

The virtual address (32 bits) is divided into three parts:
1. **First 10 bits**: Index in the Page Directory.
2. **Next 10 bits**: Index in the Page Table.
3. **Last 12 bits**: Offset within the 4 KB page.

This enables the CPU to translate virtual addresses into physical addresses effectively.

### CR3 Register

- **CR3** (Page Directory Base Register) holds the physical address of the Page Directory.
- The CPU uses this to locate the Page Directory and perform address translation.

---

## Data Structures

### Page Table Entry
```c
struct pageTableEntry {
    uint32_t present  : 1;  // Page present in memory (0)
    uint32_t rw       : 1;  // Read-only if clear, read-write if set (1)
    uint32_t user     : 1;  // Supervisor level only if clear (2)
    uint32_t pwt      : 1;  // Controls Write-Through abilities of the page (3)
    uint32_t pcd      : 1;  // Is the 'Cache Disable' bit (4)
    uint32_t accessed : 1;  // Has the page been accessed since last refresh? (5)
    uint32_t dirty    : 1;  // Has the page been written to since last refresh? (6)
    uint32_t unused   : 3;  // Unused and reserved bits (7-11)
    uint32_t frame    : 20; // Frame address (shifted right 12 bits) (12-31)
};
```

### Page Directory Entry
```c
struct pageDirectoryEntry {
    uint32_t present  : 1;  // Page present in memory (0)
    uint32_t rw       : 1;  // Read-only if clear, read-write if set (1)
    uint32_t user     : 1;  // Supervisor level only if clear (2)
    uint32_t pwt      : 1;  // Controls Write-Through abilities of the page (3)
    uint32_t pcd      : 1;  // Is the 'Cache Disable' bit (4)
    uint32_t accessed : 1;  // Has the page been accessed since last refresh? (5)
    uint32_t waste    : 1;  // Unused (6)
    uint32_t ps       : 1;  // Page size (PS=0: uses page table address) (7)
    uint32_t unused   : 4;  // Reserved bits (8-11)
    uint32_t frame    : 20; // Frame address (shifted right 12 bits) (12-31)
};
```

### Paging Structures
```c
struct pageTableEntry {
    uint32_t first   : 7;  // Reserved bits (7-11)
    uint32_t unused  : 5;  // Unused bits
    uint32_t frame   : 20; // Frame address (shifted right 12 bits) (12-31)
} __attribute__((packed));

struct pageDirectoryEntry {
    uint32_t first   : 8;  // Reserved bits (8-11)
    uint32_t unused  : 4;  // Unused bits
    uint32_t frame   : 20; // Frame address (shifted right 12 bits) (12-31)
} __attribute__((packed));
```

---

## Initializing Paging Structures

### Page Directory and Table
We create one **Page Directory** and one **Page Table** to map virtual addresses to physical memory. Both structures are page-aligned to meet the 4 KB alignment requirement.

```c
struct pdbase {
    struct pageDirectoryEntry pd[1024];
} __attribute__((aligned(4096)));

struct pdbase {
    struct pageTableEntry pt[1024];
} __attribute__((aligned(4096)));

struct pdbase d1;
struct pdbase t1;
```

### Setting Up Page Table
This function sets up the Page Directory and Page Table to map virtual addresses to physical addresses.

```c
void setPageTable() {
    d1.pd[0].first = 0b00000111;           // Page directory flags
    d1.pd[0].frame = ((uint32_t)&t1) >> 12; // Page table base address (shifted)

    uint32_t start = 0x0;
    int i = 0;
    while (start < 0xc0000) {
        t1.pt[i].first = 0b0000111;         // Page table flags
        t1.pt[i].frame = start >> 12;      // Physical frame address (shifted)
        i++;
        start += 0x1000;                   // Increment by page size (4 KB)
    }
}
```

---

## Enabling Paging

### Loading Page Directory into CR3
The following code loads the Page Directory's base address into the CR3 register.

```c
__asm__ __volatile__ (
    "mov %0, %%eax\n\t"     // Move page_directory into EAX
    "mov %%eax, %%cr3\n\t"  // Load CR3 with page directory base address
    :                       // No output operands
    : "r" ((uint32_t)&d1)   // Input operand: Page Directory address
    : "eax"                 // Clobbered register
);
```

### Enabling Paging in CR0
To enable paging, we modify the CR0 register to set the paging bit (bit 31).

```c
__asm__ __volatile__ (
    "mov %%cr0, %%eax\n\t"       // Move CR0 into EAX
    "or $0x80000000, %%eax\n\t"  // Set the paging bit
    "mov %%eax, %%cr0\n\t"       // Write back to CR0
    :                            // No output operands
    :                            // No input operands
    : "eax"                      // Clobbered register
);
```

---

## FAQs

### Why use `__attribute__((aligned(4096)))`?
The `aligned(4096)` attribute ensures that the Page Directory and Page Table are stored at 4 KB boundaries. This alignment is required by the CPU for proper address translation.

---

