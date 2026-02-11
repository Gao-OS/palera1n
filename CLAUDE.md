# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Fork of [palera1n/palera1n](https://github.com/palera1n/palera1n) — a C-based iOS/iPadOS/tvOS jailbreaking tool for checkm8-compatible devices (A8-A11 and Apple T2), supporting iOS 15.0+ and bridgeOS 5.0+. It implements a DFU-based jailbreak pipeline using checkra1n, PongoOS, and a custom ramdisk.

**Repository**: https://github.com/Gao-OS/palera1n.git

## Repository Structure

This fork includes jbinit and loader as integrated directories for a fully self-contained build:

```
palera1n/
├── src/           # Main palera1n C source
├── jbinit/        # Builds ramdisk.dmg and binpack.dmg (forked from palera1n/jbinit)
├── loader/        # iOS/tvOS loader app in Swift (forked from palera1n/loader)
├── include/       # Header files
├── docs/          # Man pages
└── dep_root/      # Pre-built static library dependencies
```

## Build Commands

```bash
# Standard build (uses local jbinit to build ramdisk/binpack)
make

# Full local build: loader → jbinit → palera1n (requires Xcode)
make full-local-build

# Build with remote CDN dependencies (original behavior)
make USE_REMOTE_DEPS=1

# Build only jbinit (produces ramdisk.dmg and binpack.dmg)
make jbinit-build

# Build only loader (produces packages/palera1nLoader.ipa)
make loader-build
make loader-tv-build    # tvOS version
make loader-all         # both iOS and tvOS

# Development build with debug symbols
make DEV_BUILD=1

# Dev build with AddressSanitizer
make DEV_BUILD=1 ASAN=1

# Dev build with ThreadSanitizer
make DEV_BUILD=1 TSAN=1

# Build with Terminal UI support
make TUI=1

# Cross-compile for Linux
make TARGET_OS=Linux CC=x86_64-linux-musl-cc

# Clean
make clean          # palera1n only
make clean-all      # including jbinit and loader
make distclean      # everything including resources
```

### Build Requirements

**For jbinit (macOS)**:
```bash
brew install gnu-sed make ldid-procursus
```

**For loader (macOS)**:
- Xcode 15+

There is no test suite. Validation is manual on physical devices. CI (`build.yml`) verifies compilation across platforms.

## Architecture

### Jailbreak Pipeline

1. **DFU entry** (`dfuhelper.c`) — guides device into DFU mode via USB events
2. **checkra1n execution** (`exec_checkra1n.c`) — runs embedded checkra1n for checkm8 exploit
3. **PongoOS communication** (`pongo_helper.c`) — sends commands and uploads payloads (KPF, ramdisk, overlay) to PongoOS over USB
4. **Device management** (`devhelper.c`) — handles device info, recovery mode, reboot via libimobiledevice/libirecovery

### Platform Abstraction

USB access has two implementations selected at compile time:
- `usb_iokit.c` — macOS native via IOKit (default on Darwin)
- `usb_libusb.c` — Linux via libusb (also available on macOS with `USE_LIBUSB=1`)

The `USE_LIBUSB` flag and `TARGET_OS` variable control which path is compiled.

### Resource Embedding

Binary resources are embedded into the palera1n binary at build time:

| Resource | Source | Build Flag to Disable |
|----------|--------|----------------------|
| ramdisk.dmg | Built by `jbinit/` or downloaded | `NO_RAMDISK=1` |
| binpack.dmg | Built by `jbinit/` or downloaded | `NO_OVERLAY=1` |
| checkra1n | Downloaded from assets.checkra.in | `NO_CHECKRAIN=1` |
| checkra1n-kpf-pongo | Downloaded from CDN | `NO_KPF=1` |
| Pongo.bin | Downloaded from CDN | `NO_CUSTOM_PONGO=1` |

By default, ramdisk.dmg and binpack.dmg are built locally from the `jbinit/` directory. Set `USE_REMOTE_DEPS=1` to download pre-built versions from CDN instead.

Resources can be overridden at runtime with `-r`, `-K`, `-o`, `-k`, `-i` flags.

### Thread Model

The tool uses pthreads with two main worker threads (`dfuhelper_thread`, `pongo_thread`) and mutex-protected shared state (`spin_mutex`, `found_pongo_mutex`, `ecid_dfu_wait_mutex`, `log_mutex`). Lock operations are in `lock_vars.c`.

### Key Headers

- `include/palerain.h` — main API, types (`devinfo_t`, `recvinfo_t`, `override_file_t`), USB abstraction, function declarations
- `include/paleinfo.h` — jailbreak flag definitions (palerain_flags bitmask)
- `include/tui.h` — terminal UI interface (compiled only with `TUI=1`)

## Code Conventions

- **C99 with strict POSIX compliance** (`-pedantic-errors`, `-D_POSIX_C_SOURCE=200112L`)
- Compiler warnings: `-Wall -Wextra` with `-Wno-unused-parameter -Wno-unused-variable`
- Logging via `LOG(level, fmt, ...)` macro which expands to `p1_log()` with file/line/function context
- Log levels: `LOG_FATAL` through `LOG_VERBOSE5` (0-8)
- Static linking on Linux; LTO on macOS
- Platform-specific code guarded by `#ifdef USE_LIBUSB`, `#if defined(__APPLE__)`, and Makefile conditionals

## Dependencies

All dependencies are built as static libraries and placed in `dep_root/`. The CI workflow builds them from source. Key libraries:
- libimobiledevice, libirecovery, libusbmuxd, libplist, libimobiledevice-glue — iOS device communication
- mbedtls — cryptography
- readline — line editing
- libusb (Linux only), gpm (Linux TUI only)
