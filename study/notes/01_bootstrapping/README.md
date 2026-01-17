# Phase 1: Bootstrapping

## Overview

Bootstrapping is the process of initializing the hardware and preparing the environment so that the Operating System kernel can run.

When a computer turns on, the RAM is full of garbage, the CPU is in a primitive 16-bit mode (to maintain compatibility with the 1978 Intel 8086), and there is no concept of "files" or "processes." Our goal in this phase is to navigate the CPU through its evolutionary stages until it reaches **64-bit Long Mode**, where our C kernel lives.

---

## The Core Topics

### 1. BIOS vs. UEFI

* **File:** `bios_vs_uefi.md`
* **The Conflict:**
  * **Legacy BIOS:** Simple, ancient, but restricted. It loads the first 512 bytes of the disk (MBR) into memory address `0x7C00` and jumps to it. It offers basic interrupt services (like "print character") but only in 16-bit mode.
  * **UEFI (Unified Extensible Firmware Interface):** The modern standard. It understands file systems (FAT32) and can load executable files directly. It starts the CPU in a more advanced state but is much more complex to develop for "from scratch."
* **Our Approach:** We will target **Multiboot2** (via GRUB). This allows us to support both BIOS and UEFI without writing two completely different loading mechanisms.

### 2. The Multiboot Specification

* **File:** `multiboot_header.md`
* **Context:** Writing a bootloader (the code that pulls the kernel from disk to RAM) is a project entirely separate from writing an OS.
* **Why do we need it?**
  * Most hobby OS developers use **GRUB** (Grand Unified Bootloader).
  * For GRUB to recognize our kernel binary as valid, we must embed a special "Magic Header" at the very start of our file.
  * This header tells GRUB: "I am an OS kernel. Please load me, give me a memory map, and put the machine in a sane state."

### 3. CPU Modes: The Ladder to 64-bit

* **File:** `cpu_modes.md`
* **The Problem:** You cannot simply "turn on" 64-bit mode. You must unlock it in stages.

#### A. Real Mode (16-bit)

The default state at power-on.

* **Access:** 1MB of RAM.
* **Addressing:** `Segment * 16 + Offset`.
* **Danger:** Any program can crash the whole machine easily.

#### B. Protected Mode (32-bit)

* **How to enter:** Enable the "A20 Line" (to access >1MB RAM) and load a basic **GDT** (Global Descriptor Table).
* **Access:** 4GB of RAM.
* **Feature:** Paging becomes available (but we usually wait).

#### C. Long Mode (64-bit)

The target state.

* **How to enter:** Enable **PAE** (Physical Address Extension), set the **EFER** MSR (Model Specific Register), and enable Paging.
* **Access:** 16 Exabytes of RAM (theoretically).
* **Feature:** Access to 64-bit registers (`rax`, `r8`-`r15`).

---

## Additional Essential Concepts

### 4. The Global Descriptor Table (GDT)

* **File:** `gdt_preliminary.md`
* **Context:** In 64-bit mode, the GDT is mostly ignored by the hardware, but it is **mandatory** to have one to enter the mode.
* **Why:** The CPU requires a table that defines what "Code" and "Data" look like. Even if we just make one giant segment that spans all of memory (Flat Memory Model), this table must exist, or the CPU will triple fault and reset.

### 5. The Stack (`rsp`)

* **File:** `stack_setup.md`
* **Context:** C functions use the "stack" to store local variables and return addresses.
* **Why:** When the bootloader hands control to us, the stack pointer (`rsp`) might be undefined or pointing somewhere unsafe. One of the very first Assembly instructions we write must be to point `rsp` to a safe block of reserved memory. If we don't, the first time we call a C function, we will overwrite random memory.

### 6. The A20 Line

* **File:** `a20_line.md`
* **Context:** A historical quirk from the 80s. The 20th bit of the address bus is disabled by default to emulate bugs in older processors.
* **Why:** If we don't enable it, memory addresses will "wrap around" at 1MB (e.g., writing to `1MB + 1` actually writes to address `1`). We need to turn this specific bit on to access all our RAM.

---

## The "Hello World" Goal of Phase 1

By the end of this phase, our goal is to have:

1. A compiled kernel binary.
2. A Multiboot header that GRUB accepts.
3. Assembly code that takes us from 32-bit (handover state) to 64-bit.
4. A jump to a C function named `kmain()`.
5. A clear indication (like a specific character in video memory) that we made it.
