# Phase 7: User Space (The Barrier)

## Overview

Currently, your Kernel runs in **Ring 0** (Supervisor Mode). It has unrestricted access to every hardware instruction and memory address.

Real applications (Shell, Text Editor, Calculator) must run in **Ring 3** (User Mode).
In Ring 3:

1. **Restricted Instructions:** You cannot use `CLI`, `HLT`, or write to CR3.
2. **Restricted Memory:** You cannot access addresses marked as "Kernel Only" in the Page Tables.
3. **The Goal:** If a User Mode program tries to do something illegal, the CPU throws a General Protection Fault (#GP), allowing the Kernel to kill just that process, not the whole machine.

---

## The Core Topics

### 1. Protection Rings & Privilege Levels

* **File:** `protection_rings.md`
* **Concepts:**
  * **CPL (Current Privilege Level):** Stored in the `CS` (Code Segment) register. 0=Kernel, 3=User.
  * **DPL (Descriptor Privilege Level):** Defines who is allowed to use a specific memory segment.
* **The Rule:** A program with CPL=3 cannot access a segment with DPL=0.

### 2. Entering User Mode (The `IRETQ` Trick)

* **File:** `entering_ring3.md`
* **The Problem:** You cannot just "jump" to Ring 3 code. The CPU forbids lowering privileges via a standard `JMP`.
* **The Solution:** We must trick the CPU.
  * We pretend an interrupt just happened and we are "returning" from it.
  * We manually build a **Fake Interrupt Stack Frame** on the Kernel Stack.
  * We execute `IRETQ` (Interrupt Return).
* **The Stack Layout (Critical):**
    1. **SS (User Data Segment Selector):** With RPL=3.
    2. **RSP (User Stack Pointer):** Address of the stack in User memory.
    3. **RFLAGS:** Interrupts enabled (IF bit set).
    4. **CS (User Code Segment Selector):** With RPL=3.
    5. **RIP (User Entry Point):** Address of the function to run.

### 3. The Task State Segment (TSS) Revisited

* **File:** `tss_stack_switching.md`
* **Why we need it NOW:**
  * When the CPU is in Ring 3 and an interrupt occurs (e.g., Timer or Keyboard), the CPU *must* switch to Ring 0 to handle it.
  * **The Danger:** The stack pointer (`rsp`) is currently in User Memory. The Kernel cannot use User Stack (it's unsafe).
  * **The Fix:** The CPU looks at the **TSS (RSP0 field)** to find the address of the Kernel Stack. It switches stacks *before* running the handler.
  * **Constraint:** If you don't set up the TSS correctly before jumping to Ring 3, the first interrupt that fires will Triple Fault the machine.

---

## The Interface

### 4. System Calls (The API)

* **File:** `syscalls.md`
* **Context:** A User program cannot just call a Kernel function (e.g., `kprint`). That would be a security violation.
* **Mechanism:** It must ask the Kernel to do it on its behalf.
* **Implementation:**
  * **Legacy:** `INT 0x80` (Software Interrupt). Slow.
  * **Modern:** `SYSCALL` / `SYSRET` instructions. Fast.
* **Setup:** You must write to special MSRs (Model Specific Registers) like `MSR_LSTAR` to tell the CPU where your `syscall_handler` function lives.

### 5. ELF Loading (Running Binaries)

* **File:** `elf_loader.md`
* **Context:** Users don't write code inside the kernel source tree. They compile separate programs (e.g., `hello.exe` or `hello.elf`).
* **Task:** The Kernel must:
    1. Read the file from the disk (or Initrd).
    2. Parse the **ELF Header** (Executable and Linkable Format).
    3. Check the "Magic Bytes" (`0x7F 'E' 'L' 'F'`).
    4. Map the segments (Code, Data) into the correct spots in Virtual Memory.
    5. Jump to the `Entry Point` defined in the header.

---

## The "Hello World" Goal of Phase 7

By the end of this phase, you will have two separate projects compiling:

1. **The Kernel (`os/`)**
2. **A User Program (`userland/hello.c`)**

The flow will be:

1. Kernel boots.
2. Kernel loads `hello.elf` from the Ramdisk.
3. Kernel performs the `IRETQ` drop to Ring 3.
4. User program runs:

    ```c
    int main() {
        syscall_print("Hello from User Space!");
        return 0;
    }
    ```

5. Kernel receives the syscall, prints the string, and resumes the user program.
