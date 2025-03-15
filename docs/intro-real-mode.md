---
id: intro-real-mode
title: Introduction to the Real Mode
sidebar_label: Introduction to the Real Mode
---

The x86 has three modes of operation, namely the real mode, the protected mode and a virtual mode. The real mode allows execution of programs in a 16-bit mode in a manner similar to that of 8086 processors. The protected mode runs programs in the native 32-bit mode that the 80386 was actually built for. This mode is where the programmer can access all the functionalities offered by the 80386 processor and is faster than the real mode. The virtual mode is a special mode where the processor rapidly switches between the protected mode environment and the real mode environment. This mode enables running both 16-bit programs and 32-bit programs simulataneously in a multitasking setting.

When the CPU is switched on, the default mode is the real mode. This is done so that older operating systems written for the 8086 CPU is still compatible with the new one. In order to switch to the more advanced protected mode, we must follow a sequence of steps that will be mentioned later. For now, our focus will be solely on the real mode and some of its features we are interested in. The real mode also has no support memory protection and allows any program to access all memory and execute any instruction. This is in contrast to the protected mode which offers several protection features that will be discussed later.

Recall the BIOS interrupt used to print characters to the screen in the previous section. The real mode offers basic support such as these BIOS interrupts which enables programmers to prep the machine before jumping to the protected mode. This will be our main area of interest in the real mode. We shall look at the interrupts offered by the BIOS that will help us load disk blocks to the memory in the end of this section. In the next chapter, we will delve into the memory addressing techniques in x86.