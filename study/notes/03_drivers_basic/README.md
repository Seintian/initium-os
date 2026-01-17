# Phase 3: Output & Debugging (The Eyes)

## Overview

A Kernel that cannot output text is a Kernel you cannot debug.

In application development, we take `printf` or `std::cout` for granted. In OS development, these do not exist. There is no console, no terminal window, and no font renderer.

In this phase, we build the "Voice" of the OS. We will implement two distinct ways to output data:

1. **The Screen:** For the user to see (VGA/Framebuffer).
2. **The Serial Port:** For the developer to see (Logging to a separate file/console).

---

## The Core Topics

### 1. VGA Text Mode (The "Easy" Way)

* **File:** `vga_text_mode.md`
* **Context:** Legacy x86 hardware maps a specific chunk of video memory to address `0xB8000`.
* **Mechanism:**
  * The screen is a grid of 80x25 characters.
  * Each character takes up **2 Bytes** in memory:
    * **Byte 1:** The ASCII character code.
    * **Byte 2:** The "Attribute" (Foreground color, Background color, Blink).
  * To print "A" in red, we simply write `0x41` and `0x04` to that memory address.
* **Limitation:** This is slowly dying out with UEFI, but it is the fastest way to get text on screen for a hobby OS.

### 2. The Serial Port (UART)

* **File:** `serial_ports.md`
* **Context:** The "Universal Asynchronous Receiver-Transmitter" is an ancient communication standard, usually found at I/O Port `0x3F8` (COM1).
* **Why it is CRITICAL:**
  * If your video driver crashes, the screen freezes. You see nothing.
  * The Serial Port is separate. You can configure QEMU to redirect serial output to your terminal window or a text file (`debug.log`).
  * **Golden Rule of OS Dev:** Always write your logs to Serial first. It is your "Black Box" flight recorder.

### 3. Framebuffers & GOP (The "Modern" Way)

* **File:** `framebuffer_gop.md`
* **Context:** Modern UEFI systems (and even Multiboot2) provide a **Linear Framebuffer**.
* **Mechanism:**
  * Instead of character slots, you get a giant array of pixels (e.g., `1920 * 1080 * 4` bytes).
  * You cannot just "print an 'A'". You must draw the 'A' pixel-by-pixel using a font bitmap.
* **Challenge:** Much harder to set up initially, but allows for graphics, images, and custom fonts later.

---

## The Software Layer

### 4. Implementing `printf`

* **File:** `printf_implementation.md`
* **Context:** Writing to memory `0xB8000` is raw and messy. We need a friendly function.
* **The Logic:**
    1. **Varargs:** Understanding how C handles variable arguments (`...`) using `<stdarg.h>` (or GCC builtins).
    2. **Parsing:** Scanning the string for `%` signs.
    3. **Conversion:** Algorithms to convert an integer (e.g., `123`) into a string ("123") or Hexadecimal (`0x7B`).
    4. **Backend Agnostic:** The code should format the string first, then call a generic `put_char()` function. This lets you switch from VGA to Serial easily.

---

## Hardware Theory Essentials

### 5. Port I/O vs. Memory Mapped I/O

* **File:** `io_methods.md`
* **Port I/O (PIO):**
  * Uses special CPU instructions: `inb` (in byte) and `outb` (out byte).
  * Used for legacy devices like the Serial Port, PS/2 Keyboard, and PIT Timer.
  * It runs on a separate bus from RAM.
* **Memory Mapped I/O (MMIO):**
  * The device registers are "mapped" to standard RAM addresses.
  * You read/write to them just like variables (e.g., `*video_ptr = 0xFF`).
  * Used for Framebuffers and modern devices (PCIe).

### 6. The `volatile` Keyword

* **File:** `volatile_keyword.md`
* **The Danger:** The C compiler loves to optimize. If you write:

    ```c
    *status_reg = 1;
    *status_reg = 0;
    ```

    The compiler thinks, "The first write is useless, I'll delete it."
* **The Reality:** In hardware, that "useless" write might have triggered a reset command.
* **The Fix:** We must mark all hardware pointers as `volatile` to forbid the compiler from optimizing our reads/writes.

---

## The "Hello World" Goal of Phase 3

By the end of this phase, your `kernel/main.c` should look like a real program:

```c
void kmain() {
    serial_init(); // Initialize logging
    vga_init();    // Clear screen

    log("Booting Initium-OS...\n"); // Goes to QEMU console
    printf("Welcome to Initium-OS!\n"); // Goes to Screen
    printf("Kernel loaded at address: 0x%x\n", &kmain);
}
```
