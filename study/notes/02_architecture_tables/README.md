# Phase 2: Architecture Setup (The Tables)

## Overview

Now that the OS has successfully booted and entered 64-bit mode, we need to establish the "Rules of Engagement" for the CPU.

By default, the CPU is running in a raw state. If a program tries to divide by zero, or if a hardware device presses a key, the CPU doesn't know what to do—so it usually just resets (Triple Fault).

To fix this, we must construct special data structures—**The Descriptor Tables**—that tell the hardware exactly how to handle memory segments, privilege levels, and interruptions.

---

## The Core Topics

### 1. The Global Descriptor Table (GDT)

* **File:** `gdt_structure.md`
* **The "Legacy" Baggage:**
  * In older 32-bit systems, the GDT was used for **Segmentation** (splitting RAM into small, protected chunks).
  * In 64-bit mode, the CPU mostly ignores segmentation (forcing a "Flat Memory Model" where base=0 and limit=MAX).
* **Why do we still need it?**
  * **Privilege Levels:** The GDT is still the *only* way to define the difference between **Ring 0 (Kernel Mode)** and **Ring 3 (User Mode)**.
  * **The TSS:** The CPU needs a GDT entry to find the Task State Segment (vital for interrupt handling).
  * **Compliance:** The x86_64 architecture explicitly requires a valid GDT to function.

### 2. The Interrupt Descriptor Table (IDT)

* **File:** `idt_interrupts.md`
* **The Concept:** The IDT is a lookup table (an array of 256 entries).
* **How it works:**
  * When an "event" occurs (e.g., Event #14), the CPU looks at index 14 in the IDT.
  * It finds a memory address (pointer) stored there.
  * It pauses what it's currently doing and jumps to that address to handle the event.
* **The 3 Types of Events:**
  1. **Exceptions (0-31):** CPU errors (Divide by Zero, Page Fault, Invalid Opcode).
  2. **IRQs (32-47):** Hardware requests (Keyboard press, Timer tick, Hard drive ready).
  3. **Software Interrupts (0x80):** Signals from User programs (System Calls).

### 3. Interrupt Service Routines (ISRs)

* **File:** `isr_stubs.md`
* **The Problem:** You cannot point the IDT directly to a standard C function.
  * C functions use the System V ABI (they expect certain registers to be preserved).
  * Interrupts can happen *anywhere*, even in the middle of a calculation. If your C function modifies a register (like `rax`) without saving the old value, the program will crash when the interrupt returns.
* **The Solution (The Assembly Stub):**
  * We write a small piece of Assembly for every interrupt.
  * **Step 1:** `CLI` (Clear Interrupts - stop new ones from happening).
  * **Step 2:** `PUSH ALL` (Save every single register to the stack).
  * **Step 3:** `CALL kernel_handler` (Run the C code).
  * **Step 4:** `POP ALL` (Restore the registers exactly as they were).
  * **Step 5:** `IRET` (Interrupt Return - special instruction to resume execution).

---

## Additional Essential Concepts

### 4. The Task State Segment (TSS)

* **File:** `tss_logic.md`
* **Context:** A special structure referenced by the GDT.
* **Why:** When an interrupt occurs while the CPU is in **User Mode (Ring 3)**, the stack pointer (`rsp`) is pointing to user memory. We *cannot* trust user memory (it might be invalid or malicious).
* **The Fix:** The TSS holds a "Known Good Stack" address (RSP0). When jumping from Ring 3 to Ring 0, the CPU automatically switches to this safe stack.

### 5. IDT "Magic" (The `lidt` instruction)

* **File:** `loading_tables.md`
* **Context:** Creating the C struct for the IDT is not enough. You must explicitly tell the CPU where it is.
* **Mechanism:** We create a special "Pointer Struct" (containing the Size and Address of the IDT) and use the assembly instruction `lidt [pointer]` to load it.

### 6. CPU Exceptions (The "Blue Screen" Logic)

* **File:** `exception_list.md`
* **Context:** The first 32 interrupts are reserved by Intel/AMD.
* **Critical Ones to Handle:**
  * **#GP (13) - General Protection Fault:** The generic "You did something illegal" error.
  * **#PF (14) - Page Fault:** The most common error in OS dev. Happens when you try to access memory that isn't mapped.
  * **#DF (8) - Double Fault:** Occurs when an exception happens *while trying to handle an exception*. If you don't handle this, the CPU Triple Faults and resets.

---

## The "Hello World" Goal of Phase 2

By the end of this phase, we will have a "Self-Healing" (or at least "Self-Reporting") system:

1. We intentionally insert a bug (like `int a = 1 / 0;`).
2. The CPU triggers Interrupt #0.
3. Our ISR Assembly Stub catches it and saves the state.
4. Our C handler takes over and prints: **"KERNEL PANIC: Divide by Zero at address [X]"**.
5. The system halts safely instead of rebooting endlessly.
