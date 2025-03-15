---
id: print16
title: Printing to Screen
sidebar_label: Printing to Screen
---

In this step, we shall build functions to help us with printing characters onto the screen in the real mode. These print functions are important for our further debugging purposes and hence it is imperative that we build these before proceeding further

Most motherboards today come with an additional flash memory chip loaded with a special piece of software called BIOS or Basic Input/Output Service. This software helps operating system designers with basic service routines that can be used to print to the screen and interact with the disk. The BIOS is automatically loaded into the RAM after the machine is powered on (also referred to as POST or Power-On Self Test). In this stage, we shall use the routines provided by BIOS to print two types of characters to the screen: ASCII and HEX. This will prove valuable for debugging in future stages.

In order to print to the screen, we use the interrupt number 0x10. This invokes the BIOS video interrupt. How do we tell the interrupt that the functionality we want to use is to print to the screen and what to be printed? This is specified in the `ax` register. The higher order byte `ah` is to be stored with 0x0e which tells the interrupt that we want to use the "print contents of `al` to the tty" function. The byte to be printed in then stored in `al`.

Let us look at an example and try printing HELLO to the screen.

<p class="codeblock-label">boot/boot.asm</p>

```
mov ah, 0x0e ; Specify the function to use i.e. print al to tty
mov al, 'H'
int 0x10
mov al, 'E'
int 0x10
mov al, 'L'
int 0x10
mov al, 'L'
int 0x10
mov al, 'O'
int 0x10

jmp $

times 510-($-$$) db 0
dw 0xaa55
```

Since we shall be using this function extensively in the future, we shall write the function to a different file and use it inside boot/boot.asm using the `%include` directive. We shall assume that we provide the starting address of the string to print in the register `bx`. We will then retrieve individual characters from the string, use the BIOS interrupt to print the character and increment address of `bx` until the end of string (null character) is reached.

<p class="codeblock-label">boot/print_string.asm</p>

```
print_string:
	mov ah, 0x0e ; Specify the function to use i.e. print al to tty
	print:
		mov al, [bx]
		int 0x10
		inc bx
		cmp al, 0
		je return
		jmp print
	return:
	ret
```

<p class="codeblock-label">boot/boot.asm</p>

```
mov bx, MSG_REAL_MODE
add bx, 0x7c00 ; This line will be explained in the next section
call print_string
jmp $

%include "boot/print_string.asm"

MSG_REAL_MODE db "Entered 16 bit Real Mode", 0

times 510-($-$$) db 0
dw 0xaa55
```

`make run` to see the message printed on the screen.

We shall now proceed to write another function for printing hex values to the screen.