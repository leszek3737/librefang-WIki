# Root — Cross.toml

# Cross.toml

## Overview

`Cross.toml` is the configuration file for [`cross`](https://github.com/cross-rs/cross), a zero-setup cross-compilation tool for Rust. This file defines build environment customization for two ARM64 targets: native Linux (GNU) and Android. It ensures that the correct system libraries and Docker images are used when producing binaries for these architectures.

This file is consumed at build time by `cross` (invoked via `cross build --target <triple>`) and does not contain executable Rust code. It lives at the repository root alongside `Cargo.toml`.

---

## Target Configuration

### `aarch64-unknown-linux-gnu`

This target produces standard Linux GNU binaries for ARM64 (e.g., for Raspberry Pi 4/5, ARM servers, or ARM CI runners).

The `pre-build` hook runs shell commands inside the cross Docker container **before** the Rust compilation step begins:

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `dpkg --add-architecture $CROSS_DEB_ARCH` | Enables package installation for the target architecture (e.g., `arm64`) |
| 2 | `apt-get update` | Refreshes the package index |
| 3 | `apt-get install --assume-yes libssl-dev:$CROSS_DEB_ARCH` | Installs OpenSSL development headers/libs for the target arch |
| 4 | `apt-get install --assume-yes libdbus-1-dev:$CROSS_DEB_ARCH` | Installs D-Bus development headers/libs for the target arch |

The `$CROSS_DEB_ARCH` environment variable is provided by `cross` and resolves to the correct Debian architecture string for the target triple (e.g., `arm64`). This multiarch approach allows both host and target architecture libraries to coexist in the same container.

**Why these libraries are needed:**
- **libssl-dev** — Required by Rust crates that link against OpenSSL (e.g., `openssl-sys`, `native-tls`, `reqwest` with default features).
- **libdbus-1-dev** — Required by crates that interface with the system D-Bus daemon (e.g., `dbus`, `zbus`).

### `aarch64-linux-android`

This target produces Android NDK-compatible binaries for ARM64 devices and emulators.

Rather than running pre-build hooks, it overrides the default Docker image:

```
ghcr.io/cross-rs/aarch64-linux-android:main
```

This pins the image to the `main` tag of the official `cross-rs` Android image, which includes the Android NDK and toolchain pre-configured. Using `:main` ensures the latest maintained image is pulled, but may introduce non-reproducible builds if the image changes upstream. If reproducibility is critical, consider pinning to a specific digest or tag.

---

## Usage

To build for the configured targets:

```bash
# ARM64 Linux (GNU)
cross build --target aarch64-unknown-linux-gnu

# ARM64 Android
cross build --target aarch64-linux-android
```

No additional flags are required — `cross` reads this file automatically when it is present at the workspace root.

---

## Relationship to the Codebase

This configuration supports cross-compilation scenarios where the project depends on native C libraries (OpenSSL, D-Bus). Without these pre-build hooks, the linker would fail with unresolved symbol errors when targeting `aarch64-unknown-linux-gnu`. The Android target override exists because the default `cross` image may lag behind or differ from what the project expects for NDK compatibility.

The file has no runtime effect and no direct connections to other source modules — it is purely a build-infra concern.