# Agent Instructions

## Purpose
Builds a GCC 15.3.0 cross-compiler toolchain targeting **aarch64-linux-gnu** (64-bit ARM) from a Windows/MSYS2 host.

## Sysroot Requirement (Critical)

**This toolchain requires kernel headers** for glibc to compile. Extract them from a Neo-PiOS Yocto build.

### Extract Kernel Headers from Neo-PiOS

From the Neo-PiOS repository (after Yocto build):

```bash
# Extract linux-libc-headers-dev package
cp $(ls build/tmp/deploy/ipk/*/linux-libc-headers-dev*.ipk) linux-libc-headers.ipk
ar x linux-libc-headers.ipk
mkdir kernel-headers
tar --zstd -xf data.tar.zst -C kernel-headers
mv kernel-headers ${OUTPUT_DIR}
```

This extracts headers to `${OUTPUT_DIRECTORY}/kernel-headers/`.

### Configure SYSROOT

Set in `env.conf`:
```bash
export SYSROOT='${OUTPUT_DIRECTORY}/kernel-headers'
```

**Why required**: Glibc needs kernel headers (`<linux/*.h>`, `<asm/*.h>`) for:
- System call numbers and calling conventions
- Kernel data structures (`struct stat`, `struct timespec`)
- Constants (errno values, signals)
- Type definitions (`pid_t`, `ssize_t`)

Without matching headers, glibc compilation fails.

---

## Key Files
- `env.conf` — Configuration: versions, paths, target architecture, SYSROOT
- `env.build` — Build script: downloads, extracts, and builds all components

## Build Order (Critical)
Components must be built in this exact sequence due to dependencies:
1. libiconv → 2. gmp → 3. isl → 4. mpfr → 5. mpc → 6. binutils → 7. xgcc (stage 1) → 8. glibc → 9. gcc (stage 2) → 10. gdb

## Environment
- **Host**: MSYS2/MinGW on Windows
- **Required packages** (build tools only — GMP/MPFR/MPC/ISL built from source):
  ```bash
  pacman -S mingw-w64-x86_64-gcc automake autoconf m4 flex bison wget texinfo make python zstd
  ```
- **Output**: Toolchain installed to `/e/Neo-PiOS-cross-toolchain/gcc-aarch64-linux-gnu/`

## Component Versions (env.conf)
| Component | Version |
|-----------|---------|
| GCC | 15.3.0 |
| Binutils | 2.42 |
| Glibc | 2.42 |
| GDB | 15.2 |
| GMP | 6.3.0 |
| MPFR | 4.2.2 |
| MPC | 1.3.1 |
| ISL | 0.24 |
| Libiconv | 1.17 |

## Target Architecture
- **Active**: `aarch64-linux-gnu` (64-bit ARM)
- **Disabled**: `armv7a-linux-gnueabihf` (commented in env.conf)
- **Tune flags**: `--with-arch=aarch64 --enable-fix-cortex-a53-843419`

## Artifacts
- Build dirs: `build/` (temporary, one subdirectory per component)
- Host tools: `host-tools/` (static libs for build host)
- Final toolchain: `/e/Neo-PiOS-cross-toolchain/gcc-aarch64-linux-gnu/`
- Kernel headers: `${OUTPUT_DIRECTORY}/kernel-headers/` (from Neo-PiOS)

## Modifying the Build
- **Add a component**: Define `SRC_*`, `DST_*`, `do_*()` function in `env.build`, add `job component` call at end
- **Change versions**: Update `*_VERSION` exports in `env.conf`
- **Change output path**: Modify `OUTPUT_DIRECTORY` in `env.conf`
- **Switch target**: Comment current TARGET block, uncomment alternative in `env.conf`
- **Change sysroot path**: Update `SYSROOT` in `env.conf` to point to extracted headers

## Build Time & Space
- **Disk space**: ~13 GB required (plus Neo-PiOS build for headers)
- **Build time**: ~2-4 hours depending on CPU cores

## Related Projects
- [Neo-PiOS](https://github.com/Neo-PiOS/Neo-PiOS) - Yocto-based embedded Linux for Raspberry Pi 4 (provides kernel headers)
