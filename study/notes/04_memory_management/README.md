# Phase 4: Memory Management (The Brain)

## Overview

Until now, we have been using memory manually. If we needed an integer, we declared `int i`. If we needed a stack, we just pointed `rsp` to a random large number and hoped it was safe.

This approach does not scale. We need a system to:

1. **Know** how much RAM is actually installed.
2. **Protect** the kernel from overwriting itself.
3. **Distribute** memory to programs that ask for it (`malloc`).
4. **Isolate** programs so they cannot steal each other's data.

This phase is split into two distinct layers: **Physical** (Hardware Reality) and **Virtual** (Software Illusion).

---

## The Core Topics

### 1. The Memory Map (E820 / UEFI)

* **File:** `memory_map.md`
* **The Reality:** RAM is not a contiguous block of empty space. It is a "Swiss Cheese" of holes.
  * Some areas are reserved for the BIOS/UEFI.
  * Some areas are Memory Mapped I/O (Video cards).
  * Some areas are actually usable RAM.
* **The Task:** We must query the bootloader (Multiboot2 tags) to get a map of these regions. We then "sanitize" this list to find every standard 4KB page of usable RAM.

### 2. Physical Memory Manager (PMM)

* **File:** `pmm_physical.md`
* **Role:** The "Warehouse Manager." It tracks the physical RAM sticks.
* **Unit:** It works strictly in **4096-byte (4KB) Frames**. It does not care about bytes, only pages.
* **Implementation Strategies:**
  * **Bitmap:** A giant array of bits where `1` = Used, `0` = Free. Simple, but searching for free blocks can be slow.
  * **Stack / Linked List:** Pushing addresses of free pages onto a stack. Allocation is instant (`pop`), but it requires memory to store the stack itself.
* **The API:** `void* pmm_alloc_frame()` and `void pmm_free_frame(void* frame)`.

### 3. Virtual Memory Manager (VMM) & Paging

* **File:** `vmm_paging.md`
* **Role:** The "Illusionist." It maps fake addresses (Virtual) to real addresses (Physical).
* **x86_64 Paging Structure:**
  * **PML4 (Page Map Level 4):** The top-level table.
  * **PDPT (Page Directory Pointer Table):** Level 3.
  * **PD (Page Directory):** Level 2.
  * **PT (Page Table):** Level 1.
* **Why do this?**
  * **Security:** User programs can be mapped as "Read Only" or "No Execute".
  * **Virtualization:** Every program thinks it has the entire 4GB of RAM to itself, starting at address `0x00000`. The VMM secretly maps these to different physical locations.

### 4. The Heap (Kernel Malloc)

* **File:** `heap_allocator.md`
* **Role:** The "Retailer." It takes large blocks from the PMM/VMM and cuts them into small slices for `malloc(16)` or `malloc(500)`.
* **The Challenge:** Fragmentation. If you allocate/free random sizes, you end up with tiny holes of free memory that are too small to use.
* **Implementation:**
  * **Linked List (First Fit):** A header at the start of every block says "I am size X and I am free/used."
  * **Slab Allocator:** Pre-cutting memory into specific sizes (e.g., a pool of 32-byte blocks, a pool of 64-byte blocks) for speed.

---

## Advanced Architecture Concepts

### 5. The Higher Half Kernel

* **File:** `higher_half_kernel.md`
* **The Concept:**
  * Traditional programs run at low memory addresses (e.g., `0x400000`).
  * To avoid clashing, we map the **Kernel** to the very top of the virtual address space (e.g., `0xFFFFFFFF80000000` -> -2GB).
* **The Trick:** The CPU actually executes code at the high address, but the page tables map it to the physical RAM at the low address (e.g., `1MB`).

### 6. The TLB (Translation Lookaside Buffer)

* **File:** `tlb_cache.md`
* **Context:** Walking 4 levels of page tables for every single memory access is slow.
* **Mechanism:** The CPU caches recent translations in the TLB.
* **The Gotcha:** If you change a page mapping in software, the CPU *doesn't know*. You must manually flush the TLB (using `invlpg` instruction), or weird bugs will happen where the CPU uses old, invalid mappings.

---

## The "Hello World" Goal of Phase 4

By the end of this phase, you should be able to do this:

```c
void test_memory() {
    // 1. Get a physical frame
    void* phys_ptr = pmm_alloc_frame(); 

    // 2. Map it to a virtual address
    vmm_map_page(phys_ptr, 0xDEADBEEF000); 

    // 3. Use the standard heap
    int* my_array = (int*) kmalloc(sizeof(int) * 100);
    my_array[0] = 50;

    kfree(my_array);
}
```
