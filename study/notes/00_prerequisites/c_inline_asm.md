# C Inline Assembly (x86_64) — GCC/Clang Guide for OS Development

This is an exhaustive guide to C Inline Assembly for GCC/Clang, specifically tailored for x86_64 OS development.

## C Inline Assembly (__asm__)

In Operating System development, 99% of the code can (and should) be written in C. However, C is an abstraction; it does not know about specific CPU instructions like `invlpg` (Invalidate TLB), `lidt` (Load IDT), or accessing I/O ports.

To bridge this gap without writing entire files in raw Assembly, GCC provides Inline Assembly. This allows you to embed Assembly instructions directly into your C functions.

---

## 1. The Syntax

There are two forms of inline assembly: Basic and Extended.

### 1.1 Basic Assembly

Used for simple instructions that do not take operands and do not modify variables.

```c
// Simply injects the assembly into the output
__asm__("hlt");
__asm__("cli");
```

### 1.2 Extended Assembly

The powerful version. It allows you to connect C variables to CPU registers.

```c
__asm__ volatile (
    "assembly code"
    : output operands   /* optional */
    : input operands    /* optional */
    : clobbers          /* optional */
);
```

- `volatile`: Tells the compiler not to optimize this code away or move it around. Critical for OS dev (e.g., timing loops or hardware I/O).
- `Assembly Code`: The string containing x86 instructions.
- `Output`: C variables where the result should be stored.
- `Input`: C variables to load into registers before execution.
- `Clobbers`: A list of registers/memory that this code modifies (so GCC knows to save/restore them).

---

## 2. Constraints (The "Glue")

GCC needs to know where to put your C variables (which register?). You specify this using Constraint Strings.

### Common x86_64 Register Constraints

| Constraint | Register (typical) | Description |
| :--- | :--- | :--- |
| "a" | `rax` / `eax` / `al` | Accumulator |
| "b" | `rbx` / `ebx` / `bl` | Base |
| "c" | `rcx` / `ecx` / `cl` | Counter |
| "d" | `rdx` / `edx` / `dl` | Data |
| "S" | `rsi` / `esi` | Source Index |
| "D" | `rdi` / `edi` | Destination Index |
| "r" | any GPR | Let GCC pick any free General Purpose Register |
| "q" | a, b, c, d | Any of the legacy accessible registers |

#### Other Constraints

| Constraint | Meaning |
| :--- | :--- |
| "m" | Memory |
| "i" | Immediate |
| "0", "1" | Matching (force same register as operand 0/1) |
| "=" | Write-Only |
| "+" | Read-Write |

---

## 3. Practical Examples (OS Development)

These are the actual functions you will write in your `kernel/arch/x86/io.h` or similar files.

### 3.1 Port I/O (The Classic)

Writing a byte to a hardware port (e.g., sending a command to the PIC):

```c
void outb(uint16_t port, uint8_t value) {
    __asm__ volatile (
        "outb %0, %1"      /* Instruction: outb value, port */
        : /* No outputs */
        : "a"(value),      /* Input %0: Load 'value' into 'al' ("a") */
          "Nd"(port)       /* Input %1: Load 'port' into 'dx' ("d") OR use generic constant ("N") */
        : /* No clobbers */
    );
}
```

Note: `outb` expects the value in `al` and the port in `dx`. We force GCC to use those registers.

Reading a byte from a port:

```c
uint8_t inb(uint16_t port) {
    uint8_t ret;
    __asm__ volatile (
        "inb %1, %0"       /* Instruction: inb port, ret */
        : "=a"(ret)        /* Output %0: Store result in 'ret', use 'al' ("=a") */
        : "Nd"(port)       /* Input %1: Port in 'dx' ("d") */
        : /* No clobbers */
    );
    return ret;
}
```

### 3.2 Reading Control Registers (`cr3`)

To read the current Page Directory Base address:

```c
uint64_t read_cr3(void) {
    uint64_t value;
    __asm__ volatile (
        "mov %%cr3, %0"    /* Note the double %% for actual registers */
        : "=r"(value)      /* Output %0: Put result in ANY register ("=r") */
        : /* No inputs */
        : /* No clobbers */
    );
    return value;
}
```

Important: When using operands (`%0`, `%1`), you must escape real register names with `%%` (e.g., `%%eax`, `%%cr3`).

### 3.3 The `cpuid` Instruction (Complex)

`cpuid` returns data in four registers simultaneously (`eax`, `ebx`, `ecx`, `edx`).

```c
void cpuid(int code, uint32_t* a, uint32_t* d) {
    __asm__ volatile (
        "cpuid"
        : "=a"(*a), "=d"(*d) /* Outputs: Write EAX to *a, EDX to *d */
        : "a"(code)           /* Inputs: Load 'code' into EAX before starting */
        : "ebx", "ecx"      /* Clobbers: CPUID modifies EBX and ECX */
    );
}
```

---

## 4. The "Clobber" List

The third section of the assembly block tells GCC: "I messed up these registers/memory, please restore them."

- `"cc"`: Condition Codes. If your assembly alters the flags (Zero Flag, Carry Flag), add this. Most arithmetic instructions do this.
- `"memory"`: The Memory Barrier. Tells GCC: "I may have modified memory arbitrarily." Forces GCC to write cached variables in registers back to RAM before the assembly runs and to reload them afterwards. CRITICAL for synchronization primitives like spinlocks.

Example: A Spinlock Hint

```c
static inline void cpu_relax(void) {
    /* 'rep; nop' is the assembly for the 'pause' instruction */
    /* Used in busy-wait loops to save power */
    __asm__ volatile ("rep; nop" ::: "memory");
}
```

---

## 5. Symbolic Names (Readable Syntax)

Using `%0`, `%1`, `%2` gets confusing quickly. You can name your operands.

```c
uint64_t basic_add(uint64_t a, uint64_t b) {
    uint64_t result;
    __asm__ volatile (
        "add %[src], %[dest]"
        : [dest] "=r" (result)  /* Output named "dest" */
        : [src]  "r"  (b),      /* Input named "src" */
          "0"    (a)            /* "0" means put 'a' in the same reg as output 'dest' */
    );
    return result;
}
```

---

## 6. Advanced: `asm goto` Labels

Modern GCC allows jumping from Assembly directly to a C label.

```c
void check_feature(uint64_t feature_flags) {
    __asm__ goto (
        "bt $0, %0 \n\t"     /* Bit Test bit 0 */
        "jc %l[found]"         /* Jump if carry (bit was 1) to label */
        : /* No outputs allowed in goto asm */
        : "r"(feature_flags)
        : "cc"
        : found                 /* The C label to jump to */
    );

    printf("Feature missing.\n");
    return;

found:
    printf("Feature found!\n");
}
```

---

## 7. Common Pitfalls

### 7.1 Optimization Reordering

GCC views `__asm__` blocks as black boxes. If you don't use `volatile` or memory clobbers, GCC might move your assembly instruction after the C code that depends on it.

Wrong:

```c
int* ptr = (int*)0xB8000;
*ptr = 0;              // GCC might run this...
__asm__("cli");      // ...after this!
```

Right:

```c
__asm__ volatile ("cli" ::: "memory");
```

### 7.2 The "Stack Pointer" Trap

NEVER list `rsp` or `esp` in the clobber list. GCC relies on the stack pointer to manage variables. If you manually change `rsp` inside inline assembly without restoring it perfectly within the same block, your kernel will crash immediately.

### 7.3 AT&T vs Intel Syntax

GCC uses AT&T Syntax by default (`op src, dest`).

- Register prefix: `%` (e.g., `%rax`)
- Immediate prefix: `$` (e.g., `$0x10`)
- Order: Source, Destination (Opposite of NASM/Intel!)

Example:

- Intel: `mov eax, 1`
- AT&T:  `movl $1, %eax`

You can force Intel syntax in GCC arguments (`-masm=intel`), but most headers use AT&T for portability. Stick to AT&T inside `__asm__` blocks unless you explicitly switch modes.

---

_End of guide._
