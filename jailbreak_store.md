# Jailbreak Store Configuration

This document explains where the jailbreak store/repository is configured in palera1n and how to replace it with a custom one.

## Overview

The jailbreak "store" (package manager repositories like Sileo, Zebra, Cydia) is **NOT hardcoded in palera1n itself**. Instead, the store configuration is determined by:

1. **The overlay (`binpack.dmg`)** — contains the loader app that presents package manager choices
2. **The bootstrap** — deployed after jailbreaking, contains the actual package manager and repo configs
3. **Runtime selection** — user picks a package manager via the loader app after jailbreaking

## Component Download URLs

These URLs in `src/Makefile` control what gets embedded into the palera1n binary. All dependencies are self-hosted on GitHub Releases (configured via `DEPS_TAG`, `DEPS_REPO`, `DEPS_BASE_URL` in the Makefiles):

| Component | URL | Purpose |
|-----------|-----|---------|
| ramdisk.dmg | `$(DEPS_BASE_URL)/ramdisk.dmg` | Boot ramdisk with jailbreak logic |
| binpack.dmg | `$(DEPS_BASE_URL)/binpack.dmg` | Overlay with loader app and utilities |
| checkra1n-kpf-pongo | `$(DEPS_BASE_URL)/checkra1n-kpf-pongo` | Kernel patchfinder |
| Pongo.bin | `$(DEPS_BASE_URL)/Pongo.bin` | PongoOS bootloader |
| checkra1n-* | `$(DEPS_BASE_URL)/checkra1n-{macos,linux-*}` | Checkm8 exploit binary |

Default base URL: `https://github.com/Gao-OS/palera1n/releases/download/deps-v1`

## How Stores Are Configured

### Boot Sequence

1. palera1n sends **binpack.dmg** (overlay) to PongoOS
2. Overlay is mounted at `/cores/binpack` during boot
3. The **loader app** from the overlay appears on the home screen
4. User opens loader and selects a package manager (Sileo, Zebra, etc.)
5. Loader downloads and deploys the **bootstrap** for that package manager
6. Bootstrap contains the repo configuration files

### Key Files in binpack.dmg

The overlay must contain (per man page):
- `Applications/` — contains the palera1n loader app
- `loader.dmg` — loader app image
- Shell, SSH server, CLI utilities

## How to Replace with Custom Store

### Option 1: Runtime Override (Recommended for Testing)

Use command-line flags to override components at runtime:

```bash
# Override the overlay (binpack) with custom one
palera1n -o /path/to/custom-binpack.dmg

# Override the ramdisk
palera1n -r /path/to/custom-ramdisk.dmg

# Override the KPF
palera1n -K /path/to/custom-kpf

# Can combine multiple overrides
palera1n -o /path/to/custom-binpack.dmg -r /path/to/custom-ramdisk.dmg
```

### Option 2: Override Dependency Source (For Custom Builds)

Override the dependency base URL at build time:

```bash
# Point to your own GitHub Releases or CDN
make DEPS_BASE_URL=https://your-cdn.example.com distclean all

# Or override individual variables
make DEPS_REPO=your-org/your-repo DEPS_TAG=your-tag distclean all
```

Or edit the defaults in the top-level `Makefile`:
```makefile
DEPS_TAG ?= deps-v1
DEPS_REPO ?= Gao-OS/palera1n
DEPS_BASE_URL ?= https://github.com/$(DEPS_REPO)/releases/download/$(DEPS_TAG)
```

Then rebuild:
```bash
make distclean
make
```

### Option 3: Create Custom Bootstrap

The `p1ctl bootstrap` command deploys a bootstrap from a zstd-compressed tarball:

```bash
p1ctl bootstrap /path/to/custom-bootstrap.tar.zst
```

Note: This only deploys the bootstrap itself. It does NOT add additional repositories or install additional packages automatically (you control this via your bootstrap contents).

## Creating Custom Components

### Custom Overlay (binpack.dmg)

Must contain:
- `Applications/` folder with your custom loader app
- `loader.dmg` file
- Shell (`/bin/sh` or similar)
- SSH server
- CLI utilities

The loader app is responsible for:
- Presenting package manager choices to users
- Downloading and deploying bootstraps
- Configuring repository URLs

### Custom Ramdisk

Minimum requirements:
- `/cores/ploosh` — jailbreak initialization
- `/usr/lib/dyld` — fake dyld with boot logic

### Custom Bootstrap

A zstd-compressed tarball containing:
- Package manager (dpkg, apt, etc.)
- Pre-configured repository sources in `/etc/apt/sources.list.d/` or equivalent
- Any pre-installed packages

## Architecture Diagram

```
palera1n binary
    │
    ├── Embedded: ramdisk.dmg (lzma compressed)
    ├── Embedded: binpack.dmg (overlay)
    ├── Embedded: checkra1n-kpf-pongo
    └── Embedded: checkra1n
            │
            ▼
    [Device boots with jailbreak]
            │
            ▼
    Overlay mounted at /cores/binpack
            │
            ▼
    Loader app appears on home screen
            │
            ▼
    User selects package manager
            │
            ▼
    Bootstrap downloaded & deployed  ◄── This is where repo URLs live
            │
            ▼
    Package manager configured with repositories
```

## What Are ramdisk.dmg and binpack.dmg?

These are HFS+ disk images built by the **jbinit** project, not palera1n itself.

### ramdisk.dmg (Boot Ramdisk)

A minimal 2MB HFS+ disk image that serves as the initial boot environment. It contains:

| Path | Purpose |
|------|---------|
| `/dyld` | Fake dynamic linker (fakedyld) — the main jailbreak entry point |
| `/payload` | Core jailbreak payload binary |
| `/payload.dylib` | Payload dynamic library |
| `/cores/ploosh` | Symlink to `/sbin/launchd` — triggers jailbreak init |
| `/sbin/launchd` | Symlink to `../payload` |
| `/usr/lib/dyld` | Symlink to `../../dyld` |

**Key insight**: When iOS boots with the ramdisk, it loads `/usr/lib/dyld` (the fake dyld), which then executes the jailbreak payload instead of normal system initialization.

### binpack.dmg (Overlay/Binpack)

A ~10MB HFS+ disk image mounted at `/cores/binpack` after boot. Contains:

| Path | Purpose |
|------|---------|
| `/Applications/` | Empty folder for loader app extraction |
| `/loader.dmg` | Compressed palera1n loader iOS app |
| `/tvloader.dmg` | Compressed palera1n loader tvOS app |
| `/usr/lib/systemhook.dylib` | System-wide dylib injection hook |
| `/usr/lib/universalhooks.dylib` | Universal process hooks |
| `/usr/lib/libellekit.dylib` | ElleKit tweak injection library |
| `/usr/sbin/p1ctl` | Command-line jailbreak control tool |
| `/usr/sbin/palera1nd` | palera1n daemon |
| `/Library/LaunchDaemons/*.plist` | Jailbreak daemon plists |
| `/Library/Frameworks/CydiaSubstrate.framework/` | Substrate compatibility (symlink to libellekit) |
| Shell, SSH, CLI utilities | From Procursus binpack.tar |

## How to Build ramdisk.dmg and binpack.dmg

This repository includes **jbinit** and **loader** as integrated directories (not submodules), enabling fully local builds.

### Prerequisites

**On macOS:**
```bash
# Install Xcode (required for loader and jbinit)
xcode-select --install

# Install dependencies
brew install gnu-sed make ldid-procursus
```

### Build Steps (Integrated)

From the palera1n root directory:

```bash
# Option 1: Full local build (loader → jbinit → palera1n)
make full-local-build

# Option 2: Build jbinit only (uses local loader if available, else downloads)
make jbinit-build

# Option 3: Build with CDN dependencies (original behavior)
make USE_REMOTE_DEPS=1
```

### Build Steps (Manual/Standalone)

If you prefer to build components separately:

```bash
# 1. Build the loader app (iOS)
cd loader
make PLATFORM=iphoneos PACKAGE_NAME=palera1nLoader
# Output: packages/palera1nLoader.ipa

# 2. Build the loader app (tvOS)
make PLATFORM=appletvos PACKAGE_NAME=palera1nLoaderTV
# Output: packages/palera1nLoaderTV.ipa

# 3. Build jbinit (will use loader IPAs from step 1-2)
cd ../jbinit
gmake -j$(sysctl -n hw.ncpu)
# Output: src/ramdisk.dmg, src/binpack.dmg

# 4. Build palera1n (will use ramdisk/binpack from step 3)
cd ..
make
```

### Output Files

```
jbinit/src/ramdisk.dmg   # Boot ramdisk
jbinit/src/binpack.dmg   # Overlay with loader and tools
loader/packages/palera1nLoader.ipa    # iOS loader app
loader/packages/palera1nLoaderTV.ipa  # tvOS loader app
```

### Test with Custom Build

```bash
palera1n -r jbinit/src/ramdisk.dmg -o jbinit/src/binpack.dmg
```

### Build Requirements Summary

| Dependency | Source | Purpose |
|------------|--------|---------|
| palera1nLoader.ipa | Built from `loader/` | iOS loader app |
| palera1nLoaderTV.ipa | Built from `loader/` | tvOS loader app |
| binpack.tar | GitHub Releases (`deps-v1`) | Procursus CLI tools (shell, SSH, etc.) |
| libellekit.dylib | Built from jbinit/src/ellekit/ | Tweak injection |
| AppleTVOS.sdk | Xcode | Target SDK for arm64 compilation |
| MacOSX.sdk | Xcode | Host SDK for build tools |

### Customizing for Your Own Store

With the integrated jbinit and loader directories, customizing is straightforward:

1. **Modify the loader** (`./loader/`):
   - Edit `Loader/` Swift source to change bootstrap URLs, repo configs, branding
   - The loader determines what package managers and repos users see

2. **Build everything**:
   ```bash
   make full-local-build
   ```

3. **Test**:
   ```bash
   ./src/palera1n -r jbinit/src/ramdisk.dmg -o jbinit/src/binpack.dmg
   ```

**For advanced customization** (modifying jbinit internals):
- Edit `jbinit/src/` source files for boot behavior changes
- Edit `jbinit/shared/*.plist` for LaunchDaemon configurations

## Related Projects

| Component | Location | Purpose |
|-----------|----------|---------|
| **jbinit** | `./jbinit/` | Builds ramdisk.dmg and binpack.dmg |
| **loader** | `./loader/` | iOS/tvOS loader app (packaged into binpack) |
| [palera1n/PongoOS](https://github.com/palera1n/PongoOS) | External | Boot environment |
| [ProcursusTeam/Procursus](https://github.com/ProcursusTeam/Procursus) | External | CLI tools bootstrap (binpack.tar) |

**Forked from** (upstream repositories):
- https://github.com/palera1n/jbinit
- https://github.com/palera1n/loader

## Summary

To use a custom jailbreak store:

1. **Quick test**: Use `-o` flag to override binpack.dmg at runtime
2. **Custom build**: Modify URLs in `src/Makefile` and rebuild
3. **Post-jailbreak**: Create custom bootstrap with your repo configs

The actual repository URLs are in the **bootstrap**, not in palera1n itself. The **loader app** in the overlay determines which bootstraps are available to users.
