# Phase 6: Multitasking (The Juggler)

## Overview

A CPU core can only execute one instruction at a time. Yet, modern OSs appear to run music, browsers, and terminals simultaneously.

This is an illusion created by **Time Slicing**.
The OS runs Process A for 20 milliseconds, then freezes it, runs Process B for 20ms, freezes it, and switches back. To the human eye, it looks parallel.

In this phase, we implement the machinery to create, manage, and switch between these "Processes."

---

## The Core Topics

### 1. The Process Control Block (PCB)

* **File:** `process_struct.md`
* **Concept:** If we want to "pause" a program, we must save its state somewhere. The PCB is a C struct that holds *everything* about a process.
* **The Essentials:**
  * **PID (Process ID):** Unique number (1, 102, etc.).
  * **RSP (Stack Pointer):** Where was the stack when we paused?
  * **CR3 (Page Directory):** Which memory map does this process use?
  * **State:** Is it `RUNNING`, `READY`, `BLOCKED`, or `ZOMBIE`?
  * **Registers:** Copies of `rax`, `rbx`, `rcx`, etc.
* **Analogy:** It is the "Bookmark" for the book that is the process.

### 2. Context Switching (The Magic Trick)

* **File:** `context_switch.md`
* **The Challenge:** How do you write a function that stops itself and starts someone else?
* **The "Switch" Function (Assembly):**
  * It cannot be written in C. It must be raw Assembly.
  * **Step 1:** Push all current registers (Callee-saved: `rbx`, `rbp`, `r12`-`r15`) onto the *current* stack.
  * **Step 2:** Save the current `rsp` into the *Old* PCB.
  * **Step 3:** Load the `rsp` from the *New* PCB.
  * **Step 4:** Pop the registers from the *new* stack.
  * **Step 5:** `RET`.
* **The Result:** When you execute `RET`, the CPU pops the return address off the *new* stack and "returns" into the middle of the *new* process's function, exactly where it left off.

### 3. The Scheduler (Round Robin)

* **File:** `scheduler_algo.md`
* **Role:** The "Boss." It decides who runs next.
* **Algorithm:**
  * **Round Robin (Simplest):** Keep a linked list of generic "Ready" tasks. Take the head, run it, push it to the back.
  * **Priority:** Later, you can add priorities (High vs Low).
* **Integration:** The Scheduler is usually called by the PIT Timer Interrupt (Phase 5). Every tick (e.g., 20ms), the Timer fires -> calls Scheduler -> calls Context Switch.

---

## Advanced Multitasking Concepts

### 4. Software vs. Hardware Switching

* **File:** `tss_deprecation.md`
* **Context:**
  * In 32-bit mode, x86 CPUs had "Hardware Task Switching" using the TSS. It was slow and rigid.
  * **In 64-bit Long Mode, Hardware Switching is removed.**
* **Implication:** You *must* implement Software Switching (saving registers manually) as described above. The TSS is now only used for holding the Kernel Stack pointer (RSP0).

### 5. Kernel Threads vs. User Processes

* **File:** `threads_vs_processes.md`
* **Kernel Thread:**
  * Lives in Ring 0.
  * Shares the same memory map (CR3) as the kernel.
  * Easy to create (just a function pointer and a stack).
* **User Process:**
  * Lives in Ring 3 (Protected).
  * Has its own isolated Page Directory (CR3).
  * Needs a complex loader (Phase 7) to start.
* **Strategy:** Implement Kernel Threads first. Get 2 kernel functions running in parallel before trying to run user programs.

### 6. The "Idle" Task

* **File:** `idle_task.md`
* **The Problem:** What happens if *all* your processes are waiting for something (blocked)? The CPU cannot just "stop." It must execute *something*.
* **The Solution:** We create a special "Idle Thread" that does nothing but run an infinite loop (often executing the `HLT` instruction to save power). The scheduler picks this thread only when no one else wants to run.

---

## The "Hello World" Goal of Phase 6

By the end of this phase, you will verify your multitasking with this test:

```c
void task_A() {
    while(1) {
        printf("A");
        sleep(100); // Give up CPU
    }
}

void task_B() {
    while(1) {
        printf("B");
        sleep(100); // Give up CPU
    }
}

void kmain() {
    create_thread(&task_A);
    create_thread(&task_B);

    enable_interrupts(); // Start the chaos

    while(1); // Main thread loops
}
```
