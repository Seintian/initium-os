# x86_64-elf Cross-Compiler Toolchain Guide

This is an exhaustive, step-by-step guide to building the x86_64-elf cross-compiler.

This guide is designed for a Linux (Debian/Ubuntu) host, as it is the standard for OS development. If you are on macOS (Homebrew) or Windows (WSL2), the logic is identical, but package names may vary slightly.

---

## The Toolchain: Building a Cross-Compiler

### 1. The Theory: Why do we need this?

You cannot use your system's default compiler (`gcc`) to build your Operating System.

- Host vs. Target
  - Host: Your computer (e.g., `x86_64-linux-gnu`).
  - Target: Your OS (e.g., `x86_64-elf` generic).

Your system GCC links with glibc (C standard library), start files (`crt0.o`), and shared libraries. None of these exist in your new OS.

- Freestanding Environment
  - A kernel runs in a "freestanding" environment (no OS services available).
  - We need a compiler that knows not to use standard headers (`<stdio.h>`, `<stdlib.h>`) or libraries by default.

- ABI (Application Binary Interface)
  - System GCC targets the System V ABI for Linux.
  - Our Cross-Compiler targets a generic ELF format that we control precisely via our linker script.

---

## 2. Prerequisites

Install required build tools and libraries.

### Debian / Ubuntu / WSL2

Open a terminal and run:

```bash
sudo apt update
sudo apt install build-essential bison flex libgmp3-dev libmpc-dev \
                 libmpfr-dev texinfo libisl-dev curl
```

### macOS (Homebrew)

```bash
brew install gcc gmp libmpc mpfr wget
```

---

### Windows

There are two practical ways to build the toolchain on Windows. **WSL2 (Windows Subsystem for Linux)** is the recommended path because it provides a real Linux environment and lets you follow the Linux instructions above verbatim. If you need a native-Windows workflow, use **MSYS2 / MinGW-w64** — it's more involved and requires using the MSYS2 MinGW64 shell.

#### Recommended: WSL2 (best compatibility)

- Install WSL2 and a Linux distribution (Ubuntu is recommended).
- Open the WSL shell and follow the Debian/Ubuntu instructions exactly (install packages, create `$HOME/src`, and run the same download/build steps).

Quick WSL2 install pointers (PowerShell as Administrator):

```sh
wsl --install -d ubuntu
# After reboot, open Ubuntu from the Start menu and complete initial setup
sudo apt update
sudo apt install build-essential bison flex libgmp3-dev libmpc-dev \
         libmpfr-dev texinfo libisl-dev curl
```

Then continue with the rest of this guide inside the WSL terminal (`bash`).

#### Native Windows: MSYS2 / MinGW-w64 (advanced)

If you cannot use WSL2, MSYS2 provides a POSIX-like environment and package manager (`pacman`). Use the **MSYS2 MinGW64** shell for 64-bit builds.

1. Install MSYS2: <https://www.msys2.org/>
2. Update packages and install build deps (run in *MSYS2 MinGW64* shell):

    ```bash
    # Update MSYS2 base packages first (may require closing and re-opening shell)
    pacman -Syu

    # Open a new MSYS2 MinGW64 shell, then:
    pacman -S --needed base-devel mingw-w64-x86_64-toolchain \
    mingw-w64-x86_64-gmp mingw-w64-x86_64-mpfr mingw-w64-x86_64-mpc \
    mingw-w64-x86_64-isl texinfo curl wget
    ```

3. Create the same workspace and environment variables inside the MSYS2 MinGW64 shell (use `~` as your home). Example:

    ```bash
    export PREFIX="$HOME/opt/cross"
    export TARGET=x86_64-elf
    export PATH="$PREFIX/bin:$PATH"
    mkdir -p $HOME/src && cd $HOME/src
    ```

4. Follow the same download and build steps as in the Linux sections. Note:

    - Use the MSYS2 MinGW64 shell (not the plain MSYS2 shell) so you have a native 64-bit toolchain and correct `gcc`/`make` behavior.
    - Some packages or scripts behave slightly differently on MSYS2; if a configure script fails, inspect `config.log` and ensure required `-dev` packages are installed via `pacman`.
    - Long paths and anti-virus can interfere with large builds on Windows — enable long paths (Windows 10/11) and temporarily disable aggressive AV if you hit spurious file-access errors.

5. Alternative: If cross-compiling from Windows is too brittle, consider using a lightweight Linux VM (VirtualBox) or a cloud builder image to perform the toolchain build.

## 3. Preparation

We will install the toolchain into a local directory (`$HOME/opt/cross`) to avoid modifying system binaries.

### 3.1 Environment Variables

Add these to your shell (and to `~/.bashrc` or `~/.zshrc` to persist):

```bash
export PREFIX="$HOME/opt/cross"
export TARGET=x86_64-elf
export PATH="$PREFIX/bin:$PATH"
```

### 3.2 Workspace Setup

Create a workspace for sources and builds (do not build in source directories):

```bash
mkdir -p $HOME/src
cd $HOME/src
```

---

## 4. Phase I: Binutils (Assembler & Linker)

`binutils` provides `ld` (linker) and `as` (assembler). Build this first so GCC can produce working binaries.

### 4.1 Download & Extract

Use a recent stable version (e.g., 2.43+):

```bash
curl -O https://ftp.gnu.org/gnu/binutils/binutils-2.43.tar.gz
tar -xf binutils-2.43.tar.gz
```

### 4.2 Build

```bash
mkdir build-binutils
cd build-binutils

../binutils-2.43/configure --target=$TARGET --prefix="$PREFIX" \
                           --with-sysroot --disable-nls --disable-werror

make -j$(nproc)
make install
```

Notes:

- `--target=$TARGET`: Force output format to `x86_64-elf`.
- `--with-sysroot`: Enables `sysroot` support (useful later for user-space).
- `--disable-nls`: Disable Native Language Support.
- `--disable-werror`: Prevent build stopping on warnings.

---

## 5. Phase II: GCC (The Compiler)

Build a bootstrapping GCC that provides the compiler without standard C libraries (libc).

### 5.1 Download & Extract

Pick a recent stable GCC (e.g., 14.2.0):

```bash
cd $HOME/src
curl -O https://ftp.gnu.org/gnu/gcc/gcc-14.2.0/gcc-14.2.0.tar.gz
tar -xf gcc-14.2.0.tar.gz
```

### 5.2 The "Red Zone" Patch (Critical for x86_64)

The System V x86_64 ABI defines a "Red Zone" (128 bytes below `rsp`) that functions may use without adjusting `rsp`. In kernel/interrupt contexts this is unsafe. We must compile `libgcc` without relying on the red zone.

1. Create (or open) `gcc/config/i386/t-x86_64-elf` inside the GCC tree (`$HOME/src/gcc-14.2.0/gcc/config/i386/t-x86_64-elf`) and add:

    ```makefile
    # Add -mno-red-zone to libgcc build flags
    MULTILIB_OPTIONS += mno-red-zone
    MULTILIB_DIRNAMES += no-red-zone
    ```

2. Ensure this file is included by `gcc/config.gcc`. Search for the `x86_64-*-elf*)` case and make sure it appends `i386/t-x86_64-elf` to `tmake_file`.

Example (snippet):

```bash
# in gcc/config.gcc
x86_64-*-elf*)
    tmake_file="${tmake_file} i386/t-x86_64-elf"
    ;;
```

### 5.3 Build GCC

```bash
mkdir build-gcc
cd build-gcc

../gcc-14.2.0/configure --target=$TARGET --prefix="$PREFIX" \
                        --disable-nls --enable-languages=c,c++ \
                        --without-headers

make all-gcc -j$(nproc)
make all-target-libgcc -j$(nproc)
make install-gcc
make install-target-libgcc
```

Flags explained:

- `--enable-languages=c,c++`: Build C and C++ (C is essential; C++ optional).
- `--without-headers`: Build for bare-metal (no C library present).

---

## 6. Verification

If successful, you should have `x86_64-elf-*` tools in `$PREFIX/bin`.

### Check Version

```bash
$HOME/opt/cross/bin/x86_64-elf-gcc --version
# Expect something like: x86_64-elf-gcc (GCC) 14.2.0
```

### Test Compilation

Create a tiny `test.c`:

```c
void kernel_main(void) {
    int x = 10;
}
```

Compile and inspect:

```bash
$HOME/opt/cross/bin/x86_64-elf-gcc -ffreestanding -c test.c -o test.o
$HOME/opt/cross/bin/x86_64-elf-objdump -d test.o
```

If you see x86_64 assembly output, the cross-compiler can produce object files for your target.

---

## 7. Troubleshooting

- `mpc/mpfr/gmp not found`: Ensure dependencies from Step 2 are installed.
- `compiler cannot create executables`: During cross GCC configure this may appear since we haven't built libc. Inspect `config.log` for details.
- Parallel build failures: If `make -j$(nproc)` fails obscurely, retry with a single core (`make`) to reveal the first failing command.

---

*End of guide.*
