# Phase 5: Interrupts & Input (The Nerves)

## Overview

In Phase 2, we set up the IDT to handle *Crashes* (Exceptions). Now, we need to handle *Events* (Hardware Interrupts).

Hardware devices (Keyboard, Mouse, Timer, Disk) need a way to tell the CPU: "Attention! I have data for you!"
They do this by sending an **IRQ (Interrupt Request)**.

However, the CPU cannot just "listen" to 50 devices at once. It needs a manager chip to collect these signals and forward them one by one. This is the job of the **Interrupt Controller**.

---

## The Core Topics

### 1. The 8259 PIC (Programmable Interrupt Controller)

* **File:** `pic_8259.md`
* **Context:** The legacy chip that manages interrupts. Even modern CPUs emulate it for backward compatibility.
* **The "Remapping" Problem (CRITICAL):**
  * By default, the PIC maps IRQ 0-7 to Interrupt Vector 0-7.
  * **Conflict:** The CPU uses Vector 0-7 for exceptions (e.g., #DivByZero, #DoubleFault).
  * **Result:** If the Timer fires (IRQ 0), the CPU thinks it's a Divide-by-Zero error and crashes.
  * **The Fix:** We must send commands to the PIC to "offset" the IRQs to start at Vector 32 (so IRQ 0 becomes Int 32, IRQ 1 becomes Int 33, etc.).
* **Master & Slave:** There are two PIC chips chained together. The Master handles IRQ 0-7, and the Slave handles IRQ 8-15.

### 2. The PIT (Programmable Interval Timer)

* **File:** `pit_timer.md`
* **Context:** A simple chip that ticks at a specific frequency (Oscillator).
* **Why do we need it?**
  * **The Heartbeat:** It fires IRQ 0 repeatedly (e.g., 1000 times a second).
  * **Preemptive Multitasking:** In Phase 6, we will use this "tick" to force the CPU to stop the current program and switch to another one. Without the PIT, a program could loop forever and freeze the OS.
  * **Sleep:** It allows us to implement `sleep(100)` by counting ticks.

### 3. The Keyboard (PS/2 Controller)

* **File:** `keyboard_ps2.md`
* **Context:** Before USB took over, PS/2 was the standard. Most PCs still emulate PS/2 for the keyboard even if a USB one is plugged in.
* **Mechanism:**
  * When you press a key, the keyboard sends IRQ 1.
  * The CPU jumps to your Handler function.
  * You read a byte from Port `0x60`.
* **Scancodes:**
  * The keyboard does *not* send ASCII characters (like 'A' or 'a').
  * It sends **Scancodes** (Position data). e.g., "The key at row 2, column 3 was pressed."
  * **Translation:** You must write a lookup table to convert Scancode `0x1E` -> Letter 'a'.

---

## Advanced Hardware Concepts

### 4. APIC (Advanced Programmable Interrupt Controller)

* **File:** `apic_ioapic.md`
* **Context:** The 8259 PIC is limited to single-core CPUs.
* **The Modern Way:** Multi-core CPUs use the **Local APIC** (one per core) and **I/O APIC** (global).
* **Strategy:** For a beginner OS, stick to the 8259 PIC first. It is much easier. Only move to APIC when you want to support multiple CPU cores.

### 5. Interrupt Masking

* **File:** `interrupt_masking.md`
* **Concept:** Sometimes, you are doing something critical (like modifying the memory manager) and you *cannot* be interrupted.
* **Instructions:**
  * `CLI` (Clear Interrupts): Ignores all IRQs.
  * `STI` (Set Interrupts): Re-enables IRQs.
* **The Danger:** If you `CLI` and then crash or loop forever, the machine is dead. The keyboard won't work, the timer won't fire. Nothing can save you but a hard reset.

---

## The "Hello World" Goal of Phase 5

By the end of this phase, your OS becomes interactive:

1. **Boot:** The OS starts.
2. **Wait:** You see a blinking cursor or a "Waiting for input..." message.
3. **Action:** You press keys on your physical keyboard.
4. **Reaction:** The letters appear on the screen instantly.
    * *Bonus:* Implement `Backspace` to delete characters.

**Congratulations!** You have built a fancy Typewriter.
