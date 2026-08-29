# Agent Instructions: Neo-PiOS Cross-Toolchain

## Critical Prerequisites

**Kernel headers are required before glibc build.** Without these, the build fails at Stage 4.

```bash
# Extract from Neo-PiOS Yocto build (after successful build)
cp $(ls build/tmp/deploy/ipk/*/linux-libc-headers-dev*.ipk) linux-libc-headers.ipk
ar x linux-libc-headers.ipk
mkdir -p ${SYSROOT}
tar --zstd -xf data.tar.zst -C ${SYSROOT}
```

Headers must be at `${SYSROOT}/usr/include/` (contains `linux/`, `asm/`, `asm-generic/`).

## Build Flow (Strict Order)

```
Stage 1: libiconv → gmp → isl → mpfr → mpc  (host tools, static libs)
Stage 2: binutils                            (aarch64 assembler/linker)
Stage 3: gcc stage 1 ("xgcc")                (C compiler only, no headers)
Stage 4: glibc headers                       (requires kernel headers)
Stage 5: gcc stage 2                         (full C/C++ compiler)
Stage 6: glibc                               (full C library)
Stage 7: gdb                                 (debugger)
```

**Why multi-stage**: GCC needs glibc, but glibc needs GCC. Stages 3-5 break this circular dependency.

## Key Files

| File | Purpose |
|------|---------|
| `build.conf` | Versions, target triplet, paths (`SYSROOT`, `HOST_TOOLS`) |
| `build.bash` | Main build script (800+ lines, `do_*()` functions) |
| `patches/` | MinGW compatibility patches (gmp, libiconv) |

## Build Commands

```bash
# Full build (2-4 hours)
bash build.bash

# Output location
${OUTPUT_DIRECTORY}/aarch64-linux-gnu/
├── bin/    # aarch64-linux-gnu-gcc, -gdb, -as, -ld
├── lib/    # libgcc, libstdc++, glibc, startup files
└── usr/    # target headers (kernel + glibc)
```

## MSYS2 Setup

```bash
pacman -S mingw-w64-x86_64-gcc automake autoconf m4 flex bison \
         wget patch texinfo make python zstd
```

**Runtime DLL**: `libwinpthread-1.dll` is copied to toolchain lib during build.

## Configuration (build.conf)

```bash
TARGET=aarch64-linux-gnu
TUNE="--enable-fix-cortex-a53-843419"           # GCC flag
GLIBC_TUNE="-march=aarch64 -mcpu=cortex-a53"    # glibc CFLAGS
SYSROOT="${OUTPUT_DIRECTORY}/${TARGET}"
```

## Common Pitfalls

| Issue | Solution |
|-------|----------|
| Glibc fails with missing `<linux/*.h>` | Kernel headers not extracted to `${SYSROOT}/usr/include/` |
| GCC configure fails on GMP | Missing `--with-gmp-include` and `--with-gmp-lib` flags (don't use `--with-gmp-prefix`) |
| MinGW configure test failures | Apply patches in `patches/` directory |
| `--disable-werror` removed | Builds tolerate warnings from cross-compilation |

## Architecture Notes

- **Host**: `x86_64-w64-mingw32` (Windows/MSYS2)
- **Target**: `aarch64-linux-gnu` (Cortex-A53, Raspberry Pi 4)
- **All deps built from source**: GMP, MPFR, MPC, ISL (not via pacman)
- **Static host libs**: `--enable-static --disable-shared` for stage 1

## Verification

```bash
aarch64-linux-gnu-gcc --version    # GCC 15.3.0
aarch64-linux-gnu-gdb --version    # GDB 15.2
echo 'int main(){}' | aarch64-linux-gnu-gcc -x c - -o test && file test
# Expected: PE32+ executable (console) x86-64
```
