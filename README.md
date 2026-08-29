# GCC Cross-Compiler Toolchain for AArch64

Builds a complete GCC 15.3.0 cross-compiler toolchain targeting **aarch64-linux-gnu** from a Windows/MSYS2 host.

## What This Is

A **cross-compiler** runs on one architecture (x86_64 Windows/MSYS2) but produces binaries for a different architecture (AArch64 Linux). This toolchain enables you to compile Linux ARM programs from your Windows development machine.

## Sysroot Requirement (Critical)

**This toolchain requires kernel headers** for the target system. The headers are extracted from a Neo-PiOS Yocto build.

### Extracting Kernel Headers from Neo-PiOS

From the Neo-PiOS repository (after a successful Yocto build):

```bash
# Extract the linux-libc-headers-dev package
cp $(ls build/tmp/deploy/ipk/*/linux-libc-headers-dev*.ipk) linux-libc-headers.ipk
ar x linux-libc-headers.ipk
mkdir -p ${SYSROOT}
tar --zstd -xf data.tar.zst -C ${SYSROOT}
```

This extracts the kernel headers to `${SYSROOT}`, where the toolchain build can find them.

**Note**: The `data.tar.zst` archive already contains the `./usr/include` directory structure, so extracting to `${SYSROOT}` will place headers at `${SYSROOT}/usr/include/` automatically.

### Configure the Toolchain

Set the `SYSROOT` variable in `build.conf` to point to the toolchain directory:

```bash
export SYSROOT='${OUTPUT_DIRECTORY}/aarch64-linux-gnu'
```

**Why this matters**: Glibc (Stage 4) requires kernel headers (`<linux/*.h>`, `<asm/*.h>`, `<asm-generic/*.h>`) to build. These headers define the kernel-userspace ABI:
- System call numbers and calling conventions
- Kernel data structures (`struct stat`, `struct timespec`)
- Constants (errno values, signals)
- Type definitions (`pid_t`, `ssize_t`)

Without matching headers, glibc compilation will fail.

**Kernel headers location**: Extract from Neo-PiOS Yocto build to `${SYSROOT}/usr/include/` (see [Sysroot Requirement](#sysroot-requirement-critical)).

## Why Build from Source?

This project builds **all dependencies from source** rather than using system packages:

| Benefit | Explanation |
|---------|-------------|
| **Version control** | Exact tested versions of GMP, MPFR, MPC, ISL |
| **Self-contained** | No dependency on MSYS2 package versions |
| **Reproducible** | Same toolchain anywhere, anytime |
| **No conflicts** | Isolated from system library updates |

## Component Versions

| Component | Version | Purpose |
|-----------|---------|---------|
| **GCC** | 15.3.0 | C/C++ compiler |
| **Binutils** | 2.42 | Assembler, linker, objdump |
| **Glibc** | 2.42 | C standard library for target |
| **GDB** | 15.2 | Source-level debugger |
| **GMP** | 6.3.0 | Big integer arithmetic |
| **MPFR** | 4.2.2 | Precise floating-point |
| **MPC** | 1.3.1 | Complex numbers |
| **ISL** | 0.24 | Loop optimization |
| **Libiconv** | 1.17 | Character encoding |

---

## Cross-Compiler Build Process

### The Fundamental Problem

You cannot simply compile GCC for a new target. GCC depends on:
1. A C library for the target (glibc)
2. Target headers and startup files
3. A working assembler and linker for the target

But glibc requires GCC to be built first. This is a **chicken-and-egg problem**.

### The Solution: Multi-Stage Build

The build is split into stages that break this circular dependency:

```
┌─────────────────────────────────────────────────────────────────┐
│  Stage 1: Build tools for the HOST (x86_64-w64-mingw32)         │
│  ┌─────────┐ ┌─────┐ ┌─────┐ ┌──────┐ ┌─────┐                  │
│  │libiconv │→│ gmp │→│ isl │→│ mpfr │→│ mpc │                  │
│  └─────────┘ └─────┘ └─────┘ └──────┘ └─────┘                  │
│           (installed to host-tools/)                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Stage 2: Binutils for the TARGET (aarch64-linux-gnu)           │
│  ┌──────────┐                                                    │
│  │ binutils │ → assembler & linker that understand ARM64        │
│  └──────────┘                                                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Stage 3: GCC Stage 1 ("xgcc" - C compiler only)                │
│  ┌──────┐                                                        │
│  │ xgcc │ → compiles C code, NO standard library, NO headers    │
│  └──────┘ → uses --with-newlib --without-headers                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Stage 4: Glibc headers (requires kernel headers from Neo-PiOS) │
│  ┌────────┐                                                      │
│  │ glibc  │ → provides <stdio.h>, <stdlib.h>, crt*.o, etc.      │
│  └────────┘ → installed to sysroot for target                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Stage 5: GCC Stage 2 (full C/C++ compiler)                     │
│  ┌─────┐                                                         │
│  │ gcc │ → now has headers and C library to link against        │
│  └─────┘ → enables C++, threads, shared libraries               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Stage 6: GDB debugger for target                               │
│  ┌─────┐                                                         │
│  │ gdb │ → source-level debugging of ARM64 binaries             │
│  └─────┘                                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Detailed Stage Breakdown

### Stage 1: Host Libraries (libiconv, GMP, ISL, MPFR, MPC)

**Purpose**: Build mathematical libraries that GCC needs during compilation.

**Why first**: GCC uses these libraries internally for:
- **GMP**: Arbitrary precision integer arithmetic (needed for compiler optimizations)
- **MPFR**: Precise floating-point constants and transformations
- **MPC**: Complex number support (for C99 complex types)
- **ISL**: Integer set library for loop optimizations (Graphite)
- **Libiconv**: Character set conversion (for internationalization)

**Build configuration**:
```bash
--prefix=${HOST_TOOLS}      # Install to host-tools/
--build=${HOST}             # Build for Windows/MinGW
--enable-static             # Static libraries only
--disable-shared            # No DLLs needed
```

**Output**: Static libraries in `host-tools/lib/` (libgmp.a, libmpfr.a, etc.)

---

### Stage 2: Binutils

**Purpose**: Create assembler (as) and linker (ld) that understand ARM64.

**Why before GCC**: GCC doesn't assemble or link itself — it generates assembly code and calls binutils to produce the final binary.

**Key configuration**:
```bash
--target=${TARGET}          # aarch64-linux-gnu
--with-libiconv-prefix=...  # Use libiconv from stage 1
--enable-lto                # Link-time optimization support
--enable-plugins            # GCC plugin support
--enable-gold               # Alternative linker
```

**Output**: `aarch64-linux-gnu-as`, `aarch64-linux-gnu-ld`, `aarch64-linux-gnu-objdump`, etc.

---

### Stage 3: GCC Stage 1 (xgcc)

**Purpose**: Build a minimal C compiler that can compile C code for ARM64.

**Why minimal**: At this point, there's no C library for the target. We can't link programs, only compile them.

**Key configuration**:
```bash
--with-newlib               # Use newlib (bare-metal) instead of glibc
--without-headers           # Don't look for target headers
--enable-languages=c        # C only (no C++ yet)
--disable-threads           # No threading support yet
--disable-shared            # Static only
--disable-lib*              # Disable all runtime libraries
```

**What it can do**:
- ✅ Compile C source to object files
- ✅ Link with explicit libraries
- ❌ Cannot compile programs that use `<stdio.h>`, `<stdlib.h>`, etc.

**What it's used for**: Compiling glibc in the next stage.

---

### Stage 4: Glibc

**Purpose**: Build the C standard library for the target system.

**Why here**: Now we have xgcc (stage 3) which can compile C code, and binutils (stage 2) which can assemble and link ARM64 objects.

**Prerequisite**: Kernel headers extracted from Neo-PiOS Yocto build to `${SYSROOT}/usr/include/` (see [Sysroot Requirement](#sysroot-requirement-critical)).

**Key configuration**:
```bash
--host=${TARGET}            # Cross-compile for aarch64
--with-headers=${SYSROOT}/usr/include  # Target kernel headers (REQUIRED)
CC="${TARGET}-gcc"          # Use the xgcc we just built
--disable-sanity-checks     # Skip host compatibility checks
libc_cv_*                   # Override autoconf checks for cross-build
```

**Special flags explained**:
- `libc_cv_forced_unwind=yes`: Required for cross-compile (glibc can't detect unwind support)
- `libc_cv_ssp=no`: Disable stack protector (not available yet)
- `--enable-hacker-mode`: Allow building without full toolchain

**Why kernel headers are required**: Glibc needs to know the kernel-userspace ABI:
- System call numbers and calling conventions
- Data structures (e.g., `struct stat`, `struct timespec`)
- Constants (e.g., `errno` values, `SIG*` signals)
- Type definitions (e.g., `pid_t`, `ssize_t`)

**Output**: C library installed to toolchain sysroot (`${INSTALL_DIR}/aarch64-linux-gnu/libc/`)

---

### Stage 5: GCC Stage 2 (Full Compiler)

**Purpose**: Build the complete C/C++ compiler with full library support.

**Why two stages**: Now that glibc exists, GCC can be built with:
- Full C++ support
- Threading (pthreads)
- Shared library generation
- All runtime libraries (libstdc++, libgcc_s, etc.)

**Key configuration**:
```bash
--enable-languages=c,c++    # Full language support
--enable-threads=posix      # POSIX threads (pthreads)
--enable-shared             # Build shared libraries (DLLs/.so)
--enable-default-ssp        # Stack protection by default
--with-sysroot=${SYSROOT}   # Find glibc headers and libs
```

**Runtime libraries built**:
- `libgcc_s.so` - GCC runtime (exception handling, etc.)
- `libstdc++.so` - C++ standard library
- `libgomp.so` - OpenMP support
- `libatomic.so` - Atomic operations
- `libsanitizer.*` - AddressSanitizer, ThreadSanitizer, etc.

**Output**: Complete toolchain

---

### Stage 6: GDB

**Purpose**: Build debugger that can debug ARM64 Linux programs.

**Configuration**:
```bash
--target=${TARGET}          # Debug aarch64-linux-gnu binaries
--with-python=python3       # Enable Python scripting
--enable-static             # Static build for portability
```

**Output**: `aarch64-linux-gnu-gdb` - can debug ARM64 ELF binaries

---

## Build Script Architecture

### File Structure

```
Neo-PiOS-cross-toolchain/
├── build.conf        # Configuration: versions, paths, target
├── build.bash        # Build script (this file)
├── patches/          # Custom patches for MinGW-w64 compatibility
│   ├── gmp-6.3.0-long-long-reliability.patch  # Fixes GMP configure test
│   └── libiconv-1.17-mingw-mbrtowc.patch      # Fixes libiconv mbtowc test
├── build/            # Temporary build directories (deleted after build)
├── host-tools/       # Host libraries (GMP, MPFR, etc.)
└── aarch64-linux-gnu/  # Final toolchain installation
    ├── bin/          # Cross-compiler binaries (aarch64-linux-gnu-*)
    ├── lib/          # Libraries
    └── usr/          # Target headers and libraries (kernel + glibc)
```

---

## Prerequisites

### MSYS2 Packages (Build Tools Only)

```bash
pacman -S mingw-w64-x86_64-gcc automake autoconf m4 flex bison \
         wget patch texinfo make python zstd
```

**Note**: GMP, MPFR, MPC, ISL are **built from source** by this script, not installed via pacman.

```
## Quick Start

```bash
# 1. Install prerequisites (see above)

# 2. Run the build
bash build.bash
```

---

## References

- [GCC Internals Manual](https://gcc.gnu.org/onlinedocs/gccint/)
- [Linux From Scratch - GCC](https://www.linuxfromscratch.org/lfs/view/stable/chapter05/gcc-pass1.html)
- [OSDev Wiki - GCC Cross-Compiler](https://wiki.osdev.org/GCC_Cross-Compiler)
- [GNU Toolchain for AArch64](https://developer.arm.com/Tools%20and%20Software/GNU%20Toolchain)
- [Neo-PiOS](https://github.com/funZX/Neo-PiOS) - Yocto-based embedded Linux for Raspberry Pi 4 (sysroot source)
