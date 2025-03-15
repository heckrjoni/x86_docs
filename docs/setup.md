---
id: setup
title: Installing a CPU Emulator
sidebar_label: Installing a CPU Emulator
---

For building an Operating System for the x86 architecture, we first need the architecture itself. We shall be specifically writing our operating system for the 80386 (also known an i386 or simply 386) microprocessor which has a 32-bit architecture. However most modern systems run on 64 bit architectures and hence we shall not be able to natively build our operating system on our machines. We shall therefore be using an emulator/simulator for the x86 CPU. Please note that all future references to the x86 architecture in this projects refers specifically to the i386 architecture.

We can use virtualisation software like VirtualBox to run an x86 environment. We shall however, for the purposes of this tutorial, use a different emulator that is more lightweight and fastweight called QEMU. For setup instructions on VirtualBox, skip to the bottom.

### QEMU

To install QEMU for your system:  
* Ubuntu/Debian-based systems: `sudo apt-get install qemu`
* CentOS/RHEL-based system: `sudo yum install qemu-kvm`
* Arch-Linux: `sudo pacman -S qemu`

You should now be able to run an i386 environment on your system by running `qemu-system-i386`