# Phase 8: File Systems (The Library)

## Overview

Up to this point, everything your OS does is ephemeral. If you turn off the power, the memory is wiped, and all your hard work (processes, variables, buffers) vanishes.

To make the OS useful, we need **Persistence**.
We need to read data from a disk (Hard Drive / SSD), understand the structure of that data (Filesystem), and present it to the user in a uniform way (VFS).

The philosophy of Unix-like systems is: **"Everything is a File."**
Whether you are reading a text file, reading keyboard input, or reading data from a network socket, the code should look exactly the same: `read(fd, buffer, size)`.

---

## The Core Topics

### 1. The Virtual File System (VFS)

* **File:** `vfs_abstraction.md`
* **The Problem:** There are dozens of file systems (FAT32, NTFS, Ext4) and devices (Keyboard, Serial, Pipes). We do not want to write `if (is_fat32) read_fat32()` every time we access data.
* **The Solution:** An abstraction layer.
  * We define a generic `fs_node` struct containing function pointers: `open`, `close`, `read`, `write`.
  * The VFS calls `node->read()`.
  * If `node` represents a keyboard, it calls the keyboard driver.
  * If `node` represents a FAT32 file, it calls the FAT32 driver.
* **Result:** The kernel logic doesn't care *where* the data comes from.

### 2. The Initial Ramdisk (Initrd)

* **File:** `initrd_format.md`
* **Context:** Writing a full Hard Drive driver is dangerous (you can wipe your real disk) and difficult.
* **The Shortcut:**
  * We ask GRUB to load a "module" (a small `.tar` or custom archive) into RAM along with our kernel.
  * The Kernel treats this chunk of memory as a "Virtual Disk."
* **Why start here?** It is Read-Only and essentially un-crashable. It allows you to load your first user programs (`hello.elf`) without writing a single line of hard drive driver code.

### 3. FAT32 (The "Hello World" Filesystem)

* **File:** `fat32_driver.md`
* **Why use it?** It is ancient, inefficient, and fragment-prone. BUT, it is:
  * Supported by every OS on earth (easy to debug).
  * Extremely simple to implement (Linked List of Clusters).
  * Perfect for USB sticks and EFI partitions.
* **Key Concepts:**
  * **BPB (BIOS Parameter Block):** The header at the start of the disk.
  * **FAT (File Allocation Table):** The map telling you which clusters belong to which file.
  * **Clusters:** The disk equivalent of Memory Pages (usually 4KB chunks).

---

## Hardware Layer

### 4. Disk Drivers (ATA / AHCI / NVMe)

* **File:** `disk_io_ata.md`
* **Context:** How do we physically get bytes from the spinning rust or flash chip?
* **Evolution:**
  * **PIO ATA (Legacy):** Sending bytes through CPU ports `0x1F0`-`0x1F7`. Slow, CPU intensive, but easiest to implement. **(Start here)**.
  * **AHCI (SATA):** Uses Memory Mapped I/O and DMA (Direct Memory Access). The CPU tells the disk "Put data at this address" and goes to sleep.
  * **NVMe:** The modern standard for PCIe SSDs. Very complex (Queues and Doorbell registers).

### 5. LBA (Logical Block Addressing)

* **File:** `lba_addressing.md`
* **Concept:** We don't talk to disks using "Cylinder, Head, Sector" anymore.
* **Mechanism:** We view the disk as a giant array of 512-byte blocks.
  * `read_sector(0)` = First 512 bytes.
  * `read_sector(100)` = Bytes 51,200 to 51,712.

---

## The "Hello World" Goal of Phase 8

By the end of this phase, you will achieve the final milestone of a basic OS: **Shell Scripting.**

1. You write a text file `config.txt` on your host machine:

    ```text
    welcome_msg=Initium OS v1.0
    background=blue
    ```

2. You bundle it into your disk image.
3. Your OS boots.
4. The Kernel code runs:

    ```c
    int fd = vfs_open("/etc/config.txt", O_RDONLY);
    char buf[100];
    vfs_read(fd, buf, 100);
    printf("Read config: %s", buf);
    ```

5. It successfully reads the file you created on Windows/Linux.

---

## Final Conclusion

**You have reached the end of the roadmap.**

If you implement all 8 phases, you will have a 64-bit Multitasking Operating System with a GUI (Framebuffer), User Mode programs, and a File System.

From here, the journey branches into infinity:

* **Networking Stack** (Ethernet, ARP, IP, TCP, HTTP).
* **Audio Drivers** (AC97, Intel HDA).
* **USB Stack** (UHCI, EHCI, XHCI).
* **Symmetric Multiprocessing** (Using all CPU cores).
