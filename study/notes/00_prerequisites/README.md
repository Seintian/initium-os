# Phase 0: Prerequisites & Tooling

## Overview

Before we can write a single line of Kernel C code, we must understand the environment we are working in. Unlike application development, where the Operating System manages memory, resources, and hardware abstraction for us, here **we are the hardware abstraction.**

This directory contains notes on the tools and languages required to build the foundation of **Initium-OS**.

---

## The Core Topics

### 1. Assembly Language (x86_64)

* **File:** `assembly_syntax.md`
* **Why do we need it?**
  * **The Boot Process:** The CPU starts in a primitive state (16-bit Real Mode or similar). C code cannot run immediately because there is no stack and memory is not initialized. We need Assembly to set up the environment for C.
  * **Hardware Access:** Some CPU instructions (like loading the GDT, enabling interrupts, or reading control registers) have no C equivalent.
  * **Debugging:** When the OS crashes, it dumps a register state. You must be able to read Assembly to understand *where* and *why* it crashed.
* **Key Focus:** Understanding the difference between **AT&T Syntax** (used by GCC/Gas) and **Intel Syntax** (used by NASM), and how to manipulate registers (`rax`, `rbx`, `rsp`, etc.).

### 2. C Inline Assembly (`__asm__`)

* **File:** `c_inline_asm.md`
* **Why do we need it?**
  * We want to write 99% of our OS in C for readability and maintainability.
  * However, sometimes we need to execute a specific CPU instruction *inside* a C function (e.g., reading from an I/O port to check the keyboard).
  * Inline Assembly allows us to embed Assembly logic directly into our C code.
* **Key Focus:** The GCC `__asm__ volatile` syntax, input/output operands, and "clobbered" registers.

### 3. The Toolchain (Cross-Compiler)

* **File:** `toolchain_setup.md`
* **Why do we need it?**
  * **The "Host" vs. "Target" Problem:** If you use the standard `gcc` on your computer (the Host), it assumes you are building a program to run on your current OS (Windows/Linux/macOS). It tries to link standard libraries (`glibc`, `stdio.h`) which **do not exist** in our new OS yet.
  * **Freestanding Environment:** We need a compiler configured to create code that runs on "bare metal" without an underlying OS. This is called a **Cross-Compiler** (specifically `x86_64-elf-gcc`).
* **Key Focus:** Building binutils and gcc from source to create a pristine build environment.

---

## Additional Essential Concepts

*These are topics you should add notes for as you encounter them.*

### 4. The Linker & Linker Scripts (`.ld`)

* **Proposed File:** `linker_logic.md`
* **Context:** When you compile a normal program, the OS decides where in RAM to put it. When writing an OS, **you** must decide where the kernel lives in physical memory.
* **Why:** The Bootloader usually loads the kernel at a specific address (e.g., `1MB`). If your code assumes it is at `0x0` but is loaded at `0x100000`, all your pointers will be wrong, and the OS will triple-fault immediately.

### 5. The Build System (Makefiles)

* **Proposed File:** `makefiles.md`
* **Context:** Compiling an OS involves compiling C files, assembling ASM files, linking them together, and converting them into an ISO image. Doing this manually is impossible.
* **Why:** We need a robust `Makefile` to automate the build process and handle dependencies (e.g., "If I change `header.h`, recompile `main.c`").

### 6. Emulation (QEMU / Bochs)

* **Proposed File:** `emulation_debug.md`
* **Context:** Testing your OS on real hardware every time requires rebooting your computer.
* **Why:** QEMU allows us to run the OS in a window. More importantly, it allows us to attach **GDB** (GNU Debugger) to pause the "virtual CPU" and inspect memory, which is impossible on real hardware without expensive JTAG tools.

---

## Recommended Reading Resources

* **OSDev Wiki:** The "Bible" of OS development. ([https://wiki.osdev.org](https://wiki.osdev.org))
* **Intel Software Developer Manuals:** The definitive reference for x86_64 instructions.
* **"Linkers and Loaders":** A book explaining how object files actually work.
