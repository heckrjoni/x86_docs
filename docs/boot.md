---
id: boot
title: Boot Sector Revisited
sidebar_label: Boot Sector Revisited
---

In Stage 3 of eXpOS, we learnt about the ROM code in page 0 of the memory and how XSM loads the OS startup code and runs it. Also recall how we were provided with an interface (xfs-interface) in order to load files easily into the disk (disk.xfs). However for x86, we do not have access to any such interfaces for now. The disk and its contents will have to be manually formatted and created by us.

x86 architecture expects the bootstrap code in block 0 of the disk. Now on a modern system, there could be multiple disks attached to the motherboard. This could be USB drives, hard disks, floppy disks etc. We then specify a boot order list in BIOS and the machine boots from the first disk on that list that is found to be bootable, skipping through the others that are not. In XSM, the architecture was restricted to a single disk at a time and hence the dilemma of which disk to boot from was not faced.

So how does the machine know whether a particular disk is bootable or not? This is where the boot flag comes. Any disk with block 0 having bootable code for x86 must hence have a boot flag (0xaa55) appended in the end. This lets x86 know that the particular disk is bootable.

In eXpOS, we loaded a small program which printed "HELLO WORLD". However we cannot do the same for our x86 operating system yet. This is because we are not aware of the port mappings in x86 yet. So in this stage we shall simply write a code that jumps into an infinite loop. This code will then be appended with empty content (zeroes) and have the boot flag in the end so as to make the entire binary in the end 512 bytes large. This is done as blocks are 512 bytes long in the x86 architecture.

<p class="codeblock-label">boot.asm</p>

```
jmp $ ; Infinite loop to current instruction

times 510-($-$$) db 0 ; Fill the remaining 510 bytes with zeroes 
dw 0xaa55 ; 2 bytes containing the boot flag
```

This code is compiled using `nasm`. Run `nasm -f bin boot.asm -o boot.bin`
You can then run the code using `qemu-system-i386 -hda boot.bin`
The `-hda` option specifies to use our compiled binary as a hard disk into the emulator.

You should expect to see a screen that is seemingly stuck after a `Booting from Hard Disk...` message.

Since the project will get increasingly complex as we progress, it is desired to have some code files organisation in our project directory. We shall hence put all our boot related files in a `boot/` directory. Create a new directory named `boot/` and move the boot.asm file inside it.