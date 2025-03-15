---
id: makefile
title: Using Makefile for Builds
sidebar_label: Using Makefile for Builds
---

As the stages progress, we'll find it increasingly difficult to compile, link and run the emulator. Hence it would be wise to start off with a Makefile right away so as to have scripted builds and runs. Lets us start by discussing about Makefile and the `make` tool.

* Makefile consists of a set of variables and rules.
* Rules have the following syntax:
	```
	targets: prerequisites
		command
		command
		command
	```
* When a target rule is triggered, it makes sure that the prerequisites are available. If not, it runs the rules for them (if available) and builds them before running this rule. If no rules are available for building them, it fails.
* By default, when the command `make` is run in a directory containing a Makefile, the first target rule is run by default.
* A rule by the target name of `target1` can be triggered by running `make target1`.
* `$^` is equivalent the string of prerequisites for that rule, `$<` is equivalent to the first prerequisite, and `$@` evaluates to the targets.
* `%` is a wildcard matching any number of characters. This is useful in writing common rules for building/compiling similar files.

For more information on Makefiles and make, visit <https://makefiletutorial.com/>. However do not spend too much time there for now. The above mentioned points will be sufficient for most of our stages for now.

Let us now add a Makefile to the root of the project directory

<p class="codeblock-label">Makefile</p>

```makefile
os-image.bin: boot/boot.bin
    cat $^ > $@

%.bin: %.asm
    nasm -f bin $< -o $@

run: os-image.bin
    qemu-system-i386 -hda os-image.bin

clean:
    rm -rf *.bin boot/*.bin
```

Our project directory should look like:

```text
project-root/
├── Makefile
└── boot/
	└── boot.asm
```

Run the command `make` from the project root. This runs the first target rule in the Makefile i.e. `os-image.bin`.
This rule has `boot/boot.bin` as a prerequisite. Hence it tries to build it first. `make` attempts to find another rule that tells it how to build `boot/boot.bin`. It matches the `%.bin` rule to build any .bin files. This has a prerequisite for `%.asm` which is `boot/boot.asm` in this context. Since that file is available, it runs the `nasm` command to compile the boot/boot.bin binary. Note how the special variables `$<` and `$@` are used. `make` now runs the `cat` command under `os-image.bin` rule which writes the contents of the prerequisite binaries into a single `os-image.bin` file.

Run the `make run` command from the project root to trigger the `run` rule in the Makefile. This runs the QEMU emulator with os-image.bin as the hard disk. Since the `run` rule has `os-image.bin` in its prerequisites (and will be built if not present), it is sufficient to simply run `make run` in the future.

To remove the binary files and clean your project directory, run `make clean`. The `clean` rule deletes all binary files in the project.