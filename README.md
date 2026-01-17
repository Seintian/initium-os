# initium-os

## Files Structure

```txt
initium-os/               # Root Repository
├── .git/
├── .gitignore
├── README.md             # Overview of the whole project
├── LICENSE
│
├── os/                   # THE ACTUAL OS (Clean, production code)
│   ├── Makefile          # The main build system for the OS
│   ├── build/            # OS build artifacts
│   ├── scripts/          # Linker scripts, ISO generation
│   │   └── linker.ld
│   └── src/              # The source code (same structure as before)
│       ├── arch/
│       ├── kernel/
│       ├── drivers/
│       └── lib/
│
└── study/                # THE LABORATORY (Messy, learning, testing)
    ├── notes/            # Markdown files, diagrams, PDFs
    │   ├── 01-booting.md
    │   └── 02-memory-map.md
    │
    ├── experiments/      # Tiny, independent C programs
    │   ├── pointer_test.c
    │   └── struct_packing.c
    │
    └── prototypes/       # Complex features built in user-space first
        ├── heap_allocator/ # Write your malloc here first using standard C
        │   ├── main.c
        │   └── malloc_toy.c
        └── file_system/
```
