# Other — librefang-desktop-capabilities

# LibreFang Desktop — Capabilities

## Overview

This module defines **Tauri capability configurations** that control which platform APIs the LibreFang application is permitted to invoke. Tauri's capability system is a security layer: unless a permission is explicitly listed here, the frontend and Rust backend cannot access the corresponding API at runtime.

Two capability sets are provided, split by platform family:

| File | Target Platforms | Purpose |
|---|---|---|
| `default.json` | macOS, Windows, Linux | Full desktop permission set |
| `mobile.json` | iOS, Android | Reduced set excluding desktop-only plugins |

## How Capabilities Work

Each JSON file declares:

- **`identifier`** — A unique name for the capability set, referenced by Tauri at build time.
- **`windows`** — Which application windows the permissions apply to. Both sets target `"main"`.
- **`platforms`** — Operating systems this capability set is active on. Tauri selects the matching set automatically.
- **`permissions`** — An array of permission tokens. Each token maps to a specific plugin or core API.

At build time, Tauri resolves these capability files and embeds an allowlist into the application binary. Any API call not covered by a granted permission will fail at runtime with a permission-denied error.

## Permission Reference

### Desktop Permissions (`default.json`)

```
core:default                              → Core Tauri APIs (window management, event system, etc.)
notification:default                      → System notification plugin
shell:default                             → Shell process spawning
dialog:default                            → Native file/message dialogs
global-shortcut:allow-register            → Register a global keyboard shortcut
global-shortcut:allow-unregister          → Unregister a global keyboard shortcut
global-shortcut:allow-is-registered       → Query whether a shortcut is registered
autostart:default                         → Launch application at login/boot
updater:default                           → In-app update checking and installation
```

The `global-shortcut` permissions are listed individually (`allow-register`, `allow-unregister`, `allow-is-registered`) rather than as `global-shortcut:default` to follow the principle of least privilege — only the specific operations needed are granted.

### Mobile Permissions (`mobile.json`)

```
core:default                              → Core Tauri APIs
notification:default                      → System notification plugin
dialog:default                            → Native dialogs
```

Several desktop plugins are omitted on mobile because they either have no equivalent on iOS/Android or are not bundled into the mobile build:

| Excluded Permission | Reason |
|---|---|
| `shell:default` | Shell spawning is not available on mobile platforms |
| `global-shortcut:*` | Global hotkeys are a desktop-only concept |
| `autostart:default` | Mobile OS manages app lifecycle differently; autostart is not applicable |
| `updater:default` | Mobile apps distribute updates through app stores |

## Adding a New Permission

When integrating a new Tauri plugin or core API:

1. **Check platform compatibility.** If the plugin only works on desktop, add the permission token to `default.json` only. If it works everywhere, add it to both files.
2. **Use the narrowest token available.** Prefer `plugin:allow-<action>` over `plugin:default` when you only need specific operations.
3. **Validate against the schema.** Both files reference the Tauri JSON schema at the `$schema` key. Your editor can use this for autocomplete and validation.

## File Locations

```
librefang-desktop/capabilities/
├── default.json    # Desktop (macOS, Windows, Linux)
└── mobile.json     # Mobile (iOS, Android)
```

These files are consumed by the Tauri build pipeline — no runtime imports or code references are needed.