# initium-os

## Files Structure

```txt
repo-name/
├── .git/
├── .gitignore
├── README.md           # Documentation and build instructions
├── LICENSE             # MIT or GPL are common for OS projects
├── Makefile            # The main build entry point
├── build/              # (Ignored) Intermediate object files (.o) go here
├── dist/               # (Ignored) Final binaries (.iso, .bin) go here
├── docs/               # Notes on memory maps, hardware specs, etc.
├── scripts/            # Helper scripts (e.g., creating ISOs, GDB hooks)
│   ├── linker.ld       # The Linker Script (CRITICAL for OS dev)
│   └── make_iso.sh
├── src/
│   ├── arch/           # Architecture specific code
│   │   └── x86_64/     # (Or i386/arm64)
│   │       ├── boot/   # Assembly boot code (Multiboot header, start.S)
│   │       └── lib/    # Arch-specific implementations (e.g., memcpy optimized)
│   ├── drivers/        # Hardware drivers
│   │   ├── vga/        # Screen printing
│   │   ├── serial/     # Serial port (essential for debugging logging)
│   │   └── keyboard/
│   ├── include/        # Header files (.h)
│   │   ├── kernel/
│   │   └── drivers/
│   ├── kernel/         # The core kernel logic (Architecture independent)
│   │   ├── main.c      # Kernel entry point
│   │   ├── memory.c    # Pmm/Vmm (Physical/Virtual memory managers)
│   │   └── panic.c     # Kernel panic handler
│   └── lib/            # Standard library replacements (libc-like functions)
│       ├── string.c    # strlen, memcpy, memset implementation
│       └── stdio.c     # printf implementation
└── tests/              # Unit tests for libc functions
```
