---

id: tss_preq1  
title: Segment Selectors and Privilege Levels in x86 (32-bit)  
sidebar_label: Prerequisite 1  

---



The x86 architecture uses segmentation to provide memory protection and controlled access to different memory areas. Segment selectors and privilege levels are key elements of this system.

---

## **Segment Selector**

A segment selector is a 16-bit value used to access a segment descriptor in the Global Descriptor Table (GDT) or Local Descriptor Table (LDT). The segment descriptor defines the base address, size, and attributes of a memory segment.

### **Format of Segment Selector**

```
15         3   2       0
+-----------+---+-------+
| Index     | TI| RPL   |
+-----------+---+-------+
```

- **Index (13 bits):** Specifies the index of the descriptor in the GDT or LDT.  
- **TI (Table Indicator, 1 bit):**
  - `0`: Selects the GDT.  
  - `1`: Selects the LDT.  
- **RPL (Requested Privilege Level, 2 bits):**  
  Indicates the privilege level requested by the selector.  

---

## **Descriptor Privilege Level (DPL)**

- DPL is a 2-bit field in the segment descriptor.  
- Defines the minimum privilege level required to access the segment.  
- **Lower DPL values indicate higher privilege:**  
  - `0`: Most privileged (Kernel).  
  - `3`: Least privileged (User).  

---

## **Current Privilege Level (CPL)**

- The CPL reflects the privilege level of the currently executing code.  
- Stored in the `CS` (Code Segment) register.  
- Determines access to other segments and resources:  
  - CPL is compared against the DPL of a segment descriptor when accessing it.  

---

## **Requested Privilege Level (RPL)**

- RPL is part of the segment selector.  
- Represents a privilege level assigned to a segment for temporary access control.  
- Used to impose additional restrictions when accessing segments.  

---

## **Privilege Checks**

When accessing a segment, the following privilege checks occur:  

1. **CPL ≤ DPL:**  
   The current privilege level must be less than or equal to the descriptor's privilege level.  
   
2. **Max(CPL, RPL) ≤ DPL:**  
   The maximum of CPL and RPL must be less than or equal to the descriptor's privilege level.  

--- 
