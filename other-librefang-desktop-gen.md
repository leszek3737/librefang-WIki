# Other — librefang-desktop-gen

# librefang-desktop/gen — Tauri Generated Output

## Overview

The `gen/` directory holds **auto-generated Tauri build artifacts** — platform scaffolding and security schema definitions. It is not hand-written source code. The files here are produced by Tauri's CLI during `cargo tauri build`, `cargo tauri android init`, or `cargo tauri ios init`, and they should generally not be edited directly.

## Directory Layout

```
gen/
├── android/
│   └── README.md          # Placeholder — populated by `cargo tauri android init`
├── apple/
│   └── README.md          # Placeholder — populated by `cargo tauri ios init`
└── schemas/
    ├── acl-manifests.json  # Full ACL manifest for every Tauri plugin used
    ├── capabilities.json   # Active capability definitions (desktop + mobile)
    └── desktop-schema.json # JSON Schema for validating capability files
```

## Platform Scaffolding

### `android/`

Populated by running `cargo tauri android init` from `crates/librefang-desktop/`. Once initialized, this directory contains the Gradle-based Android Studio project that wraps the Tauri webview for Android builds.

### `apple/`

Populated by running `cargo tauri ios init` (macOS only) from `crates/librefang-desktop/`. Produces the Xcode project for iOS builds.

Both directories are empty placeholders until the respective `init` commands are run. They are typically excluded from version control or committed as stubs.

## Security Schemas

The `schemas/` directory is the more substantive part of this module. It defines **what the frontend is allowed to do** through Tauri's capability-based security model.

### `acl-manifests.json`

Machine-readable manifest of every permission exposed by every Tauri plugin bundled into LibreFang. Each top-level key is a plugin identifier, containing:

| Field | Purpose |
|---|---|
| `default_permission` | The permission set granted when you reference `plugin-name:default` |
| `permissions` | Individual `allow-*` / `deny-*` entries, each controlling a specific IPC command |
| `global_scope_schema` | JSON Schema for scoped permissions (e.g., which shell commands or file paths are allowed) |

**Plugins covered:**

- **`autostart`** — boot auto-start (enable, disable, is_enabled)
- **`core`** — umbrella for all `core:*` sub-plugins
- **`core:app`** — app metadata, theming, listeners
- **`core:event`** — listen, unlisten, emit, emit_to
- **`core:image`** — image creation from bytes/path, RGBA extraction
- **`core:menu`** — full menu CRUD (append, prepend, insert, remove, popup, etc.)
- **`core:path`** — path manipulation (resolve, join, dirname, basename, etc.)
- **`core:resources`** — resource handle closing
- **`core:tray`** — system tray CRUD and configuration
- **`core:webview`** — webview lifecycle, positioning, zoom, devtools
- **`core:window`** — window state queries and mutations (over 60 commands)
- **`dialog`** — native dialogs (message, open, save)
- **`global-shortcut`** — keyboard shortcut registration
- **`notification`** — OS notifications with channel support (Android)
- **`shell`** — process spawning and URL opening, with scoped command allowlists
- **`updater`** — self-update workflow (check, download, install)

### `capabilities.json`

The **active security policy** for LibreFang. Defines two capability profiles:

#### `default` — Desktop

```json
{
  "identifier": "default",
  "windows": ["main"],
  "local": true,
  "platforms": ["macOS", "windows", "linux"],
  "permissions": [
    "core:default",
    "notification:default",
    "shell:default",
    "dialog:default",
    "global-shortcut:allow-register",
    "global-shortcut:allow-unregister",
    "global-shortcut:allow-is-registered",
    "autostart:default",
    "updater:default"
  ]
}
```

Key points:
- Applies only to the `"main"` window label on desktop platforms
- `core:default` pulls in all core sub-plugin defaults (path, event, window, webview, app, image, resources, menu, tray)
- `global-shortcut` is **not** using the `default` set (which grants nothing) — instead, individual `allow-register`, `allow-unregister`, `allow-is-registered` permissions are explicitly listed
- `shell:default` grants URL opening (`http(s)`, `tel:`, `mailto:`) but not arbitrary command execution

#### `mobile` — iOS / Android

```json
{
  "identifier": "mobile",
  "windows": ["main"],
  "local": true,
  "platforms": ["iOS", "android"],
  "permissions": [
    "core:default",
    "notification:default",
    "dialog:default"
  ]
}
```

Deliberately excludes desktop-only plugins:
- No `shell` (no process spawning on mobile)
- No `global-shortcut` (no global keyboard shortcuts on mobile)
- No `autostart` (not applicable on mobile)
- No `updater` (updates go through app stores)

### `desktop-schema.json`

A JSON Schema (Draft 7) that validates capability files. Useful for:
- IDE autocompletion when editing capability JSON files
- CI validation that capability files conform to expected structure
- Understanding the full set of valid permission identifiers and their scopes

The schema encodes conditional logic — for example, `shell:*` permission entries can carry an `allow`/`deny` scope that defines which shell commands are permitted, using the `ShellScopeEntry` schema.

## How to Regenerate

From the workspace root or `crates/librefang-desktop/`:

```bash
# Regenerate schemas (happens automatically during build)
cargo tauri build

# Initialize Android project
cd crates/librefang-desktop
cargo tauri android init

# Initialize iOS project (macOS only)
cargo tauri ios init
```

## Modifying Permissions

To change what the frontend can access:

1. Edit the capability files in the Tauri source configuration (typically `src-tauri/capabilities/` or inline in `tauri.conf.json`), **not** the generated `gen/schemas/` files directly.
2. Run `cargo tauri build` or `cargo tauri dev` to regenerate the schemas.
3. The `gen/schemas/` files will be overwritten with the updated policy.

## Relationship to the Rest of the Codebase

```
tauri.conf.json / capabilities/
        │
        ▼
   gen/schemas/          ◄── Auto-generated, do not hand-edit
        │
        ├── acl-manifests.json   → Referenced by Tauri runtime for IPC access control
        ├── capabilities.json    → Loaded at app startup to gate frontend→backend calls
        └── desktop-schema.json  → Used by dev tooling / IDE support
```

The frontend (in the companion webview crate) calls Tauri IPC commands. The runtime checks every call against the compiled capabilities from this directory. If a permission is not listed, the call is rejected at runtime with a security error.