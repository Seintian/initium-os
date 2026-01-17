# The Definitive x86_64 Assembly Guide for Operating System Development

**Target Architecture:** AMD64 / Intel 64\
**Syntax:** NASM (Intel Syntax)\
**Scope:** Bare Metal (Ring 0) to User Land (Ring 3)

---

## 1. Register Reference (64-bit Mode)

In Long Mode (64-bit), general-purpose registers are 64 bits wide. `pusha` and `popa` are **removed**.

### General Purpose

| 64-bit | 32-bit | 16-bit | 8-bit (Low) | 8-bit (High) | Usage / ABI Convention (System V) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `rax` | `eax` | `ax` | `al` | `ah` | Return Value / syscall number |
| `rbx` | `ebx` | `bx` | `bl` | `bh` | Callee Saved |
| `rcx` | `ecx` | `cx` | `cl` | `ch` | 4th Argument / Loop Counter |
| `rdx` | `edx` | `dx` | `dl` | `dh` | 3rd Argument / I/O Port |
| `rsi` | `esi` | `si` | `sil` | - | 2nd Argument / Source Ptr |
| `rdi` | `edi` | `di` | `dil` | - | 1st Argument / Dest Ptr |
| `rbp` | `ebp` | `bp` | `bpl` | - | Stack Frame Base (Callee Saved) |
| `rsp` | `esp` | `sp` | `spl` | - | Stack Pointer |
| `r8` | `r8d` | `r8w` | `r8b` | - | 5th Argument |
| `r9` | `r9d` | `r9w` | `r9b` | - | 6th Argument |
| `r10` | `r10d` | `r10w` | `r10b` | - | Caller Saved / Static Chain |
| `r11` | `r11d` | `r11w` | `r11b` | - | Caller Saved / Temp |
| `r12` | `r12d` | `r12w` | `r12b` | - | Callee Saved |
| `r13` | `r13d` | `r13w` | `r13b` | - | Callee Saved |
| `r14` | `r14d` | `r14w` | `r14b` | - | Callee Saved |
| `r15` | `r15d` | `r15w` | `r15b` | - | Callee Saved |

> **CRITICAL:** Writing to the lower 32-bits (e.g., `mov eax, 1`) **zeros the upper 32-bits** of the register.

### Control Registers (System State)

* **CR0:** `PE` (Bit 0 - Protected Mode), `PG` (Bit 31 - Paging).
* **CR2:** Contains the **Virtual Address** that caused a Page Fault (#PF).
* **CR3:** Physical address of the root paging structure (PML4).
* **CR4:** `PAE` (Bit 5 - Physical Address Ext), `PGE` (Bit 7 - Page Global), `OSFXSR` (Bit 9 - SSE).

### Model Specific Registers (MSRs)

Read via `rdmsr`, write via `wrmsr`. (Address in `ecx`, value in `edx:eax`).

* **EFER (0xC0000080):** Extended Feature Enable. Bit 8 (`LME`) enables Long Mode. Bit 11 (`NXE`) enables No-Execute.
* **STAR (0xC0000081):** Syscall Target Address.
* **LSTAR (0xC0000082):** Target RIP for `syscall`.
* **FS_BASE (0xC0000100):** Base address for `fs` segment (Thread Local Storage).
* **GS_BASE (0xC0000101):** Base address for `gs` segment (Kernel Structures).
* **KERNEL_GS_BASE (0xC0000102):** Used by `swapgs` instructions.

---

## 2. Boot Protocol: The Trampoline

You cannot boot directly into 64-bit mode. You must transition: **16-bit (Real) -> 32-bit (Protected) -> 64-bit (Long).**

### 2.1 The Global Descriptor Table (GDT)

In 64-bit mode, the GDT is still required, but Base and Limit are ignored for CS/DS/ES/SS.

* **CS (Code Segment):** Must have the "Long Mode" bit set (Bit 53) and "Present" bit.
* **Null Descriptor:** Index 0 must be null.

#### GDT Definition (NASM)

```nasm
align 8
gdt_start:
    dq 0x0000000000000000 ; Null Descriptor

.code: equ $ - gdt_start
    ; Access: Present(1), Ring0(00), Code/Data(1), Exec(1), Conf(0), Read(1), Acc(0) -> 0x9A
    ; Flags: Granularity(1), 64-bit(1), 32-bit(0) -> 0xA
    dw 0xFFFF       ; Limit (ignored)
    dw 0x0000       ; Base (ignored)
    db 0x00         ; Base (ignored)
    db 10011010b    ; Access Byte
    db 10101111b    ; Flags (Low 4) / Limit (High 4)
    db 0x00         ; Base

.data: equ $ - gdt_start
    ; Access: Present(1), Ring0(00), Code/Data(1), Exec(0), Dir(0), Write(1), Acc(0) -> 0x92
    ; Flags: 0xC (Note: 64-bit flag is 0 for data segments)
    dw 0xFFFF
    dw 0x0000
    db 0x00
    db 10010010b    ; Access Byte
    db 11001111b    ; Flags
    db 0x00

gdt_descriptor:
    dw $ - gdt_start - 1
    dq gdt_start
```

### 2.2 Enabling Long Mode (Code Sequence)

Assumes we are in 32-bit Protected Mode with Paging currently disabled.

```nasm
bits 32

enter_long_mode:
    ; 1. Set up Page Tables (See Section 3)
    ; Assume PML4 is at address 0x1000 and properly linked
    mov eax, 0x1000
    mov cr3, eax

    ; 2. Enable PAE (Physical Address Extension) in CR4
    mov eax, cr4
    or eax, 1 << 5
    mov cr4, eax

    ; 3. Set Long Mode Enable (LME) in EFER MSR
    mov ecx, 0xC0000080
    rdmsr
    or eax, 1 << 8
    wrmsr

    ; 4. Enable Paging (PG) in CR0
    mov eax, cr0
    or eax, 1 << 31
    mov cr0, eax

    ; 5. Load 64-bit GDT
    lgdt [gdt_descriptor]

    ; 6. Far Jump to flush pipeline and load CS
    jmp 0x08:long_mode_start

bits 64
long_mode_start:
    ; 7. Update segment registers
    mov ax, 0x10 ; Data segment offset
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    
    ; 8. Setup Stack
    mov rsp, 0x200000
    
    ; KERNEL IS NOW LIVE IN 64-BIT MODE
```

---

## 3. Memory Management: 4-Level Paging

x86_64 uses a 48-bit Virtual Address space mapped via 4 levels.

**Virtual Address Structure:**

```txt
[ Sign Ext (16b) | PML4 (9b) | PDPT (9b) | PD (9b) | PT (9b) | Offset (12b) ]
```

### 3.1 Paging Tables

Every table (PML4, PDPT, PD, PT) contains 512 entries. Each entry is 64 bits (8 bytes).

### 3.2 Page Table Entry (PTE) Flags

These flags apply to entries in all levels.

| Bit | Name | Description |
| :--- | :--- | :--- |
| 0 | P | Present (1=Map, 0=Fault) |
| 1 | RW | Read/Write (1=Write, 0=Read Only) |
| 2 | US | User/Supervisor (1=User/Ring3, 0=Kernel/Ring0) |
| 3 | PWT | Write Through |
| 4 | PCD | Cache Disable |
| 5 | A | Accessed (Set by CPU) |
| 6 | D | Dirty (Set by CPU on write - PT only) |
| 7 | PS | Page Size (1=Huge Page in PD/PDPT) |
| 63 | NX | No Execute (Prevents code execution from data pages) |

### 3.3 Helper Instructions

* `invlpg [addr]`: Invalidates TLB for specific address.
* `mov cr3, rax`: Flushes entire TLB (Performance heavy).

---

## 4. Interrupt Descriptor Table (IDT)

The IDT tells the CPU how to handle Exceptions and Hardware Interrupts.

In 64-bit, the IDT entry size is 16 Bytes (128 bits).

### 4.1 IDT Entry Structure

```nasm
; 0-15:   Offset Low
; 16-31:  Selector (Kernel Code Segment)
; 32-34:  IST (Interrupt Stack Table index, 0-7)
; 35-39:  Reserved
; 40-43:  Type (0xE = Interrupt Gate, 0xF = Trap Gate)
; 44:     0
; 45-46:  DPL (Descriptor Privilege Level)
; 47:     Present
; 48-63:  Offset Middle
; 64-95:  Offset High
; 96-127: Reserved
```

### 4.2 Interrupt Handler (ISR) Template

Since `pusha` is gone, you must manually save context. The CPU automatically pushes: `SS`, `RSP`, `RFLAGS`, `CS`, `RIP` (and optional Error Code).

```nasm
extern interrupt_handler_c

global isr_stub
isr_stub:
    ; Stack on entry: [SS, RSP, RFLAGS, CS, RIP, (Error?)]

    ; 1. Save Register Context (System V ABI volatile + others)
    push r15
    push r14
    push r13
    push r12
    push r11
    push r10
    push r9
    push r8
    push rbp
    push rdi
    push rsi
    push rdx
    push rcx
    push rbx
    push rax

    ; 2. Prepare C Argument
    mov rdi, rsp  ; Pass stack pointer as first arg to C handler

    ; 3. Call Kernel Logic
    call interrupt_handler_c

    ; 4. Restore Context
    pop rax
    pop rbx
    pop rcx
    pop rdx
    pop rsi
    pop rdi
    pop rbp
    pop r8
    pop r9
    pop r10
    pop r11
    pop r12
    pop r13
    pop r14
    pop r15

    ; 5. Clear Error Code (if present - logic needed here to know if it exists)
    add rsp, 8 

    ; 6. Return
    iretq
```

---

## 5. System Calls (syscall / sysret)

The `int 0x80` method is slow. Modern OSs use `syscall`.

### 5.1 Initialization

You must configure MSRs to enable syscall.

```nasm
init_syscalls:
    ; 1. Enable SCE (System Call Extensions) bit in EFER
    mov ecx, 0xC0000080
    rdmsr
    or eax, 1         ; Bit 0
    wrmsr

    ; 2. Load STAR (0xC0000081)
    ; Bits 32-47: Kernel CS
    ; Bits 48-63: User CS base (Segments are loaded as UserCS+8, UserDS+16)
    mov ecx, 0xC0000081
    mov edx, 0x00130008 ; Example: UserBase=0x13, KernelBase=0x08
    mov eax, 0x00000000
    wrmsr

    ; 3. Load LSTAR (0xC0000082) - The Target RIP
    mov ecx, 0xC0000082
    mov rax, syscall_entry_point
    mov rdx, rax
    shr rdx, 32
    wrmsr

    ; 4. Load FMASK (0xC0000084) - RFLAGS mask
    ; Usually we mask out Interrupts (bit 9) so syscalls enter with CLI
    mov ecx, 0xC0000084
    mov eax, 0x200    ; Clear IF (Interrupt Flag)
    mov edx, 0
    wrmsr
    ret
```

### 5.2 The Syscall Entry Point

The `syscall` instruction does not switch stacks automatically. You must use `swapgs` to find the kernel stack.

```nasm
global syscall_entry_point
syscall_entry_point:
    ; On entry: 
    ; RIP -> RCX
    ; RFLAGS -> R11
    ; Stack is still USER STACK (Dangerous!)

    swapgs                  ; Switch to Kernel GS (Points to Per-CPU data)
    mov [gs:0x0], rsp       ; Save User RSP to scratch space
    mov rsp, [gs:0x8]       ; Load Kernel RSP

    push r11                ; Save RFLAGS
    push rcx                ; Save Return RIP

    ; ... Save other regs ...

    ; Handle Syscall
    ; Args: RDI, RSI, RDX, R10, R8, R9
    ; Note: RCX is used for RIP, so 4th arg is R10, not RCX!

    ; ... Restore regs ...

    pop rcx
    pop r11

    mov rsp, [gs:0x0]       ; Restore User RSP
    swapgs                  ; Restore User GS

    o64 sysret              ; Return to Ring 3
```

---

## 6. Concurrency & Synchronization

For Multi-core (SMP) OSs, atomicity is required.

### 6.1 Atomic Primitives

* `lock`: Prefix for instructions (add, sub, inc, dec, xor, etc.) to make them atomic.
* `xchg`: Atomically swaps two operands.
* `cmpxchg`: Compare and Exchange (CAS).

### 6.2 Spinlock Implementation

```nasm
; void acquire_lock(uint64_t* lock_addr);
global acquire_lock
acquire_lock:
    lock bts qword [rdi], 0  ; Bit Test and Set bit 0. Atomic.
    jnc .success             ; If Carry=0, bit was 0, now 1. We have lock.
    
.spin:
    pause                    ; Hint to CPU that we are spinning (saves power)
    test qword [rdi], 1      ; Check if lock is free (non-atomic peek)
    jnz .spin                ; If still 1, keep spinning
    jmp acquire_lock         ; Retry atomic operation

.success:
    ret

; void release_lock(uint64_t* lock_addr);
global release_lock
release_lock:
    mov qword [rdi], 0       ; Write 0. Stores are atomic on x86 aligned addresses.
    ret
```

---

## 7. Context Switching

The core mechanism of a Scheduler.

```nasm
; void switch_task(Context** old, Context* new);
global switch_task
switch_task:
    ; RDI = Pointer to address where we save old stack
    ; RSI = Address of new stack

    ; 1. Save Callee-Saved Registers of OLD task
    push rbx
    push rbp
    push r12
    push r13
    push r14
    push r15
    pushfq ; Save flags if needed

    ; 2. Save RSP to the old thread struct
    mov [rdi], rsp

    ; 3. Load RSP from the new thread struct
    mov rsp, rsi

    ; 4. Restore Callee-Saved Registers of NEW task
    popfq
    pop r15
    pop r14
    pop r13
    pop r12
    pop rbp
    pop rbx

    ret ; Returns into the new task's code
```

---

## 8. Miscellaneous Hardware I/O

### Port I/O (Legacy Hardware)

Required for PIC, PIT, PS/2 Keyboard.

```nasm
; uint8_t inb(uint16_t port);
global inb
inb:
    mov dx, di    ; Port in DX
    in al, dx     ; Result in AL
    ret

; void outb(uint16_t port, uint8_t value);
global outb
outb:
    mov dx, di    ; Port in DX
    mov ax, si    ; Value in AX (lower byte used)
    out dx, al
    ret
```

### Wait for I/O

Useful for older hardware that requires a delay between port writes.

```nasm
io_wait:
    out 0x80, al  ; Write to unused port 0x80
    ret
```
