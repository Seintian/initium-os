# Initium-OS Development Roadmap

## Phase I: The Bootstrapping (Entering the Machine)

*Before writing kernel logic, the hardware must accept the code.*

- [ ] **Boot Protocol Decision**: Choose between Legacy BIOS (Multiboot) or UEFI.
- [ ] **Multiboot Header**: Implement the "Magic" numbers required by GRUB.
- [ ] **Real Mode (16-bit)**: Understanding the CPU entry state (if writing a custom bootloader).
- [ ] **Protected Mode (32-bit)**: Setting up the initial jump to access 4GB+ memory.
- [ ] **Long Mode (64-bit)**: Enabling PAE and EFER MSR to reach 64-bit capability.
- [ ] **Stack Setup**: Manually pointing the Stack Pointer (`rsp`) to a safe memory block.

## Phase II: Architecture Setup (The Tables)

*Establishing the rules for memory access and error handling.*

- [ ] **GDT (Global Descriptor Table)**: Define code/data segments for Kernel and User modes.
- [ ] **IDT (Interrupt Descriptor Table)**: Create the lookup table for CPU exceptions and hardware interrupts.
- [ ] **ISRs (Interrupt Service Routines)**: Write Assembly stubs to save CPU state before handling interrupts.
- [ ] **Exception Handlers**: Implement C functions to catch crashes (e.g., Divide by Zero, Page Fault).

## Phase III: Output & Debugging (The Eyes)

*Visualizing what is happening inside the machine.*

- [ ] **VGA Text Mode (0xB8000)**: Basic string printing for legacy BIOS.
- [ ] **Framebuffer / GOP**: Pixel-perfect graphics output for UEFI/Modern rendering.
- [ ] **Serial Port (UART/COM1)**: Implement a driver to send logs to the host machine (vital for debugging).
- [ ] **`printf` Implementation**: Write a custom string formatting function.

## Phase IV: Memory Management (The Brain)

*Managing RAM usage to prevent the OS from overwriting itself.*

- [ ] **Memory Map Parsing**: Read the e820/UEFI memory map to identify available RAM.
- [ ] **PMM (Physical Memory Manager)**:
  - [ ] Bitmap or Stack implementation to track free physical pages.
- [ ] **VMM (Virtual Memory Manager)**:
  - [ ] Setup Paging (PML4, PDPT, PD, PT).
  - [ ] Map the Higher Half Kernel.
- [ ] **Heap Allocator**: Implement `malloc` and `free` (Linked List or Slab Allocator).

## Phase V: Interrupts & Input (The Nerves)

*Making the system interactive.*

- [ ] **PIC (Programmable Interrupt Controller)**: Remap IRQs to avoid collision with CPU exceptions.
- [ ] **APIC / IO-APIC**: (Advanced) Setup for multi-core interrupt handling.
- [ ] **PIT (Programmable Interval Timer)**: Configure the system clock/heartbeat.
- [ ] **Keyboard Driver**:
  - [ ] PS/2 Controller setup.
  - [ ] Scancode to ASCII translation.

## Phase VI: Multitasking (The Juggler)

*Running multiple processes simultaneously.*

- [ ] **PCB (Process Control Block)**: Define the structure for a process (Registers, Stack, Page Table).
- [ ] **Context Switching**: Assembly logic to swap register states between processes.
- [ ] **Scheduler**: Implement an algorithm (Round Robin) to pick the next task.
- [ ] **TSS (Task State Segment)**: Configure stack switching for User Mode -> Kernel Mode transitions.

## Phase VII: User Space (The Barrier)

*Protecting the kernel from user programs.*

- [ ] **Ring 3 Transition**: Logic to drop CPU privileges from Ring 0 to Ring 3.
- [ ] **System Calls (syscall/sysret)**: Create the API for user programs to talk to the kernel.
- [ ] **ELF Loader**: Write a parser to load executable binaries into memory.

## Phase VIII: File Systems (The Library)

*Persistent storage management.*

- [ ] **VFS (Virtual File System)**: Abstract interface for files/devices.
- [ ] **Initrd (Initial Ramdisk)**: Load a small filesystem into memory at boot.
- [ ] **Disk Driver**: Read/Write sectors to a hard drive (ATA/AHCI/NVMe).
- [ ] **Filesystem Driver**: Implement FAT32 or Ext2 logic.
