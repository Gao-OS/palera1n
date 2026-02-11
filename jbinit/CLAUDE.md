# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

jbinit (formerly plooshInit) is the ramdisk initialization system for the palera1n jailbreak. It produces two disk images:
- **ramdisk.dmg** — bootstrapping ramdisk that replaces dyld, hooks launchd, and initializes the jailbreak environment during early boot
- **binpack.dmg** — overlay filesystem containing jailbreak binaries, libraries, LaunchDaemons, and the palera1n loader apps

All binaries target **arm64-appletvos** (minimum tvOS 12.0) but run on iOS/iPadOS/tvOS/bridgeOS via dyld platform patching at boot.

## Build Commands

```bash
# Full build (ramdisk.dmg + binpack.dmg)
gmake -j$(sysctl -n hw.ncpu)

# Development build (no optimization, debug symbols, debug shell)
gmake DEV_BUILD=1 -j$(sysctl -n hw.ncpu)

# With AddressSanitizer (increases ramdisk from 2MB to 8MB)
gmake DEV_BUILD=1 ASAN=1 -j$(sysctl -n hw.ncpu)

# Clean build artifacts
gmake clean        # source artifacts only
gmake distclean    # including tools
```

### Build Prerequisites (macOS)

```bash
brew install gnu-sed make ldid-procursus
```

Xcode must be installed (provides AppleTVOS SDK, macOS SDK, clang). The build uses `gmake` (GNU Make) — the macOS default `make` is BSD Make and will not work.

### External Dependencies

Before building, these files must be placed in `src/`:
- `binpack.tar` from `https://static.palera.in/binpack.tar`
- `palera1nLoader.ipa` and `palera1nLoaderTV.ipa` from palera1n CDN

The parent palera1n Makefile handles downloading these automatically when building from the top level.

## Architecture & Boot Chain

See `OVERVIEW.md` for the full narrative. The execution order is:

1. **fakedyld** (`src/fakedyld/`) — Replaces `/usr/lib/dyld` via kernel patch. Bare-metal C (`-nostdlib -ffreestanding`) using a bundled libc. Mounts filesystems, patches dyld for multi-platform support, then executes real launchd.

2. **payload_dylib** (`src/payload_dylib/`) — Injected into `/sbin/launchd` via `DYLD_INSERT_LIBRARIES`. Hooks launchd with ellekit/Dobby to inject systemhook into all spawned processes and load additional LaunchDaemons.

3. **payload** (`src/payload/`) — Multi-call binary (also `p1ctl`, `palera1nd`, `jailbreakd`). Runs three boot stages:
   - **prelaunchd** (`-f`): Mount binpack/loader DMGs, setup fakefs
   - **sysstatuscheck** (`-s`): Remount filesystems, generate SSH keys, run rc.d scripts
   - **launchdaemons** (`-j`): uicache apps, spawn daemons

4. **systemhook** (`src/systemhook/`) — Injected into most processes. Loads tweaks, re-injects on child spawn, fixes `__builtin_available` for dyld-patched binaries.

5. **universalhooks** (`src/universalhooks/`) — Injected into specific daemons for platform-specific behavior (rootless path redirection, tvOS app enablement, watchdog hooking).

6. **bridgehook** (`src/bridgehook/`) — BridgeOS-specific hooks using Dobby (since ellekit doesn't support bridgeOS).

## Source Structure

| Directory | Purpose | Notes |
|-----------|---------|-------|
| `src/fakedyld/` | Early boot dyld replacement | Bare-metal, no stdlib, ~4K LOC |
| `src/payload/` | Multi-call binary (payload/p1ctl/palera1nd/jailbreakd) | Main runtime logic |
| `src/payload/loader/` | Boot stage implementations | prelaunchd, sysstatuscheck, launchdaemons |
| `src/payload/jailbreakd/` | Privileged daemon | Root helper, userspace reboot |
| `src/payload/p1ctl/` | CLI tool | User-facing jailbreak management |
| `src/payload_dylib/` | launchd injection dylib | Hooks via ellekit/Dobby |
| `src/systemhook/` | Process-wide injection | Tweak loading, spawn hooking |
| `src/universalhooks/` | Daemon-specific hooks | Platform abstraction |
| `src/bridgehook/` | BridgeOS hooks | Uses Dobby framework |
| `src/libjailbreak/` | Shared library | XPC IPC, BMHash, platform detection |
| `src/mount_cores/` | Mount utility | Platform-versioned variants |
| `src/ellekit/` | Substitute framework (submodule) | Swift, Xcode build |
| `src/libs/` | TBD stubs | IOKit, APFS, MobileGestalt, etc. |
| `tools/` | Build tools | libdmg-hfsplus (HFS+ DMG creation) |
| `shared/` | Runtime resources | LaunchDaemon plists, boot image, licenses |
| `include/` | Headers | paleinfo.h, libjailbreak.h, payload.h, mount_args.h |

## Key Data Structures

**`struct paleinfo`** (`include/paleinfo.h`) — Boot parameters appended to `/dev/md0`:
- `magic` (`'PLSH'`), `version` (2), `kbase`, `kslide`, `flags` (64-bit bitmask), `rootdev`

**`palerain_option_t` flags** — 64-bit bitmask controlling jailbreak behavior (rootful/rootless, fakefs, safemode, verbose, etc.). Shared between palera1n and jbinit.

## Build System Details

The build is orchestrated by 23+ Makefiles in a hierarchical dependency chain:

```
Root Makefile → apple-include (SDK header prep) → tools (libdmg-hfsplus)
             → src/Makefile → libjailbreak.a → all SUBDIRS → ramdisk.dmg → binpack.dmg
```

`SUBDIRS` = `fakedyld payload_dylib payload systemhook universalhooks mount_cores ellekit bridgehook`

Each subdirectory has its own Makefile. All subdirectories depend on `libjailbreak` being built first. The `apple-include` target copies and patches macOS/tvOS SDK headers to remove availability restrictions (`__IOS_PROHIBITED`, `__API_UNAVAILABLE`, etc.).

DMG images are created using libdmg-hfsplus tools (`hfsplus`, `dmg`) built from `tools/`. On macOS with Xcode, `hdiutil` is used for some operations instead.

## Code Conventions

- **C99/GNU17** with `-pedantic-errors`, `-Wall -Wextra`
- fakedyld is **freestanding** (`-nostdlib -nostdlibinc -ffreestanding`) with its own libc from Embedded Artistry
- Production builds use `-Oz -flto=full`; dev builds use `-O0`
- Logging: `LOG(level, fmt, ...)` macro → `p1_log()` with file/line/function context
- Error checking: `CHECK_ERROR(action, loop, msg, ...)` macro with errno handling
- Hooking: `DYLD_INTERPOSE(_replacement, _replacee)` for dyld interposition
- Symbol visibility: `-fvisibility=hidden` for dylibs, `SHOOK_EXPORT` for exported symbols
- Path abstraction: `JB_ROOT_PATH(path)` for rootless/rootful compatibility
- Code signing: `ldid` (Procursus) for ad-hoc signing with entitlements

## Testing

There is no automated test suite. Validation is manual on physical devices. The `tools/patch_dyld-test` target provides a standalone test harness for dyld patching logic only.
