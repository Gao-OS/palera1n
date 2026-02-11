# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Swift-based iOS/tvOS bootstrap installer for the palera1n jailbreak (forked from palera1n/loader). After a device is jailbroken, this app installs the bootstrap environment and a package manager (e.g., Sileo, Zebra). It communicates with the `palera1nd` daemon over XPC to perform privileged operations.

## Build Commands

```bash
# iOS build → packages/palera1nLoader.ipa
make

# tvOS build → packages/palera1nLoaderTV.ipa
make PLATFORM=appletvos PACKAGE_NAME=palera1nLoaderTV

# Clean
make clean
```

The Makefile copies macOS SDK headers into `apple-include-{platform}/`, strips `__IOS_PROHIBITED`/`__TVOS_PROHIBITED` annotations with `gsed`, builds with `xcodebuild`, signs with `ldid`, and packages as IPA. Requires `gnu-sed` and `ldid-procursus` (`brew install gnu-sed ldid-procursus`).

You can also build directly in Xcode via the `Loader.xcworkspace` (needed to include NimbleKit).

There is no test suite. Validation is manual on jailbroken devices. Use [AppSync Unified](https://github.com/akemin-dayo/AppSync) for on-device debugging.

## Architecture

### Navigation Flow

```
AppDelegate → SceneDelegate → LRTabbarController
                                ├── LRBootstrapViewController  (Bootstrap tab)
                                └── LRSettingsViewController   (Settings tab)
```

`LRBootstrapViewController` fetches config from `palera.in/loaderv2.json`, presents available package managers, then launches `LRStagedViewController` which drives a three-phase install: download → bootstrap deploy → package install.

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **LREnvironment** | `Utilities/Environment/` | Central singleton: process spawning (posix_spawn as root via persona override), path prefixing (rootful vs rootless), bootstrap status detection, system actions (reboot/respring/uicache) |
| **LRBootstrapper** | `Utilities/Bootstrap/` | Downloads bootstrap tar + debs, deploys via XPC, installs packages with dpkg. Uses delegate pattern for status callbacks |
| **JailbreakD** | `Utilities/XPC/` | Swift wrapper over C XPC client (`jailbreakd.c/h`). Talks to `in.palera.palera1nd.systemwide` Mach service for `DeployBootstrap`, `ObliterateJailbreak`, `GetPinfoFlags`, etc. |
| **LRConfig** | `Utilities/Config/Models/` | Decodable model for remote JSON config: platform-specific bootstrap URLs, manager lists, repos, version requirements |
| **NimbleKit** | `NimbleKit/` | Local Swift Package (4 libraries): reusable UI base classes (`LRBaseStagedViewController`, `LRBaseStructuredTableViewController`), extensions, JSON fetch service, transition animations |

### Rootful vs Rootless Path Prefixing

All filesystem paths go through `String.jb_prefix()` (rootless → `/var/jb` prefix) or `String.binpack()` (always → `/cores/binpack` prefix). The mode is determined by jailbreak flags from `JailbreakD.getFlags()`.

### C Interop

A bridging header (`Loader-Bridging-Header.h`) exposes:
- `jailbreakd.c` — XPC protocol implementation
- `nvram.c` — NVRAM read/write
- Private framework headers (LSApplicationWorkspace, MobileGestalt)
- `dyld_get_active_platform()` for runtime platform detection

### Platform Conditionals

```swift
#if os(iOS) / #else          // iOS vs tvOS UI differences
#if targetEnvironment(simulator)  // Simulator stubs for JailbreakD (returns mock flags)
```

## Build Configuration

`Configuration/Loader.xcconfig` defines compile-time constants:
- `DOTFILE_PATH` = `/.installed_palera1n` (bootstrap detection marker)
- `CONFIG_URL` = `palera.in/loaderv2.json` (remote config endpoint)
- `SYSTEM_HEADER_SEARCH_PATHS` points to the generated `apple-include-{platform}` directory

## Code Conventions

- **Indentation**: tabs, width 4 (see `.editorconfig`)
- **Class prefix**: `LR` for all app types
- **Private methods**: underscore prefix (`_load()`, `_showManagerPopup()`)
- **File splitting**: extensions in separate files named `ClassName+feature.swift`
- **Async pattern**: `Task.detached` + `await MainActor.run` for UI updates
- **Error display**: `UIAlertController.showAlert()` extension, errors shown to user rather than thrown silently
- **Platforms**: iOS 15+ and tvOS 15+, arm64 only for device builds
