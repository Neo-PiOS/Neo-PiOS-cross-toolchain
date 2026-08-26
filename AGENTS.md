# Agent Instructions

## Purpose
Builds a GCC 15.3.0 cross-compiler toolchain targeting **aarch64-linux-gnu** (64-bit ARM) from a Windows/MSYS2 host.

## Sysroot Requirement (Critical)

**This toolchain requires a pre-built sysroot** with kernel headers for glibc to compile.

### Source: Neo-PiOS Project

The sysroot is built using [Neo-PiOS](https://github.com/funZX/Neo-PiOS), a Yocto/OpenEmbedded-based embedded Linux distribution for Raspberry Pi 4.

**Build command** (from Neo-PiOS repository, in Yocto environment):
```bash
bitbake core-image-minimal -c populate_sdk
```

**Sysroot location**:
```
build/tmp/work/<machine>-neopios-linux/core-image-minimal/<version>/sdk/sysroots-components/aarch64-neopios-linux/
```

### Configure SYSROOT

Set in `env.conf`:
```bash
export SYSROOT='/path/to/yocto/sysroots/aarch64-neopios-linux'
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
- **Output**: Toolchain installed to `/e/build-win32-arm-toolchain/gcc-aarch64-linux-gnu/`

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
- Final toolchain: `/e/build-win32-arm-toolchain/gcc-aarch64-linux-gnu/`

## Modifying the Build
- **Add a component**: Define `SRC_*`, `DST_*`, `do_*()` function in `env.build`, add `job component` call at end
- **Change versions**: Update `*_VERSION` exports in `env.conf`
- **Change output path**: Modify `OUTPUT_DIRECTORY` in `env.conf`
- **Switch target**: Comment current TARGET block, uncomment alternative in `env.conf`
- **Change sysroot**: Update `SYSROOT` to point to different Yocto build

## Build Time & Space
- **Disk space**: ~13 GB required (plus ~10-20 GB for Neo-PiOS sysroot build)
- **Build time**: ~2-4 hours depending on CPU cores

## Related Projects
- [Neo-PiOS](https://github.com/funZX/Neo-PiOS) - Yocto-based embedded Linux for Raspberry Pi 4 (provides sysroot)
