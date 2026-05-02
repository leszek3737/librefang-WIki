# Other — librefang-desktop-capabilities

# librefang-desktop-capabilities

## Overview

This module defines the **Tauri capability configurations** for LibreFang. Capabilities are a core Tauri security mechanism — they declare which plugins and core APIs the application is allowed to use, which windows they apply to, and which platforms they target. Without an explicit capability entry, a Tauri plugin API is blocked at runtime.

The module contains two capability files that split permissions by platform family:

| File | Target Platforms |
|---|---|
| `default.json` | macOS, Windows, Linux |
| `mobile.json` | iOS, Android |

## Why Two Files?

Several Tauri plugins bundled in LibreFang are **desktop-only** — they have no implementation on mobile operating systems. Including their permissions in a mobile capability would cause build failures or runtime errors. The split ensures each platform family only references plugins that are actually compiled into its binary.

**Desktop-only plugins excluded from mobile:**
- `shell` — process spawning
- `global-shortcut` — system-wide hotkeys
- `autostart` — login-item / launch-at-login
- `updater` — in-app update flow

## Capability Structure

Every capability file shares the same Tauri schema:

```json
{
  "$schema": "https://raw.githubusercontent.com/nicedoc/tauri/refs/heads/dev/crates/tauri-utils/schema.json",
  "identifier": "<unique-name>",
  "description": "<human-readable summary>",
  "windows": ["main"],
  "platforms": ["..."],
  "permissions": ["..."]
}
```

| Field | Purpose |
|---|---|
| `$schema` | JSON Schema for IDE validation and autocomplete |
| `identifier` | Unique name referenced by Tauri at build time |
| `description` | Human-readable documentation of the capability's intent |
| `windows` | Which webview windows these permissions apply to (`main` is the primary window) |
| `platforms` | OS targets — Tauri ignores the capability on non-matching platforms |
| `permissions` | Ordered list of permission identifiers granted to the window |

## Permission Reference

### Desktop (`default.json`)

| Permission | Plugin | Grants |
|---|---|---|
| `core:default` | Tauri core | Standard core APIs (window management, events, etc.) |
| `notification:default` | `notification` | Sending OS-level notifications |
| `shell:default` | `shell` | Spawning child processes and executing shell commands |
| `dialog:default` | `dialog` | Native file pickers, message dialogs |
| `global-shortcut:allow-register` | `global-shortcut` | Registering a system-wide keyboard shortcut |
| `global-shortcut:allow-unregister` | `global-shortcut` | Removing a previously registered shortcut |
| `global-shortcut:allow-is-registered` | `global-shortcut` | Querying whether a shortcut is currently registered |
| `autostart:default` | `autostart` | Launching the app at user login |
| `updater:default` | `updater` | Checking for and applying in-app updates |

### Mobile (`mobile.json`)

| Permission | Plugin | Grants |
|---|---|---|
| `core:default` | Tauri core | Standard core APIs |
| `notification:default` | `notification` | Push / local notifications |
| `dialog:default` | `dialog` | Native dialogs (alerts, action sheets) |

## Architecture

```mermaid
graph TD
    subgraph Tauri Security Layer
        Cap["Capabilities Module"]
    end

    subgraph Desktop Plugins
        Core["core:default"]
        Notif["notification:default"]
        Shell["shell:default"]
        Dialog["dialog:default"]
        GS["global-shortcut:*"]
        Auto["autostart:default"]
        Up["updater:default"]
    end

    subgraph Mobile Plugins
        MCore["core:default"]
        MNotif["notification:default"]
        MDialog["dialog:default"]
    end

    Cap -->|"default.json"| Desktop Plugins
    Cap -->|"mobile.json"| Mobile Plugins
```

## How to Add a New Permission

1. **Identify the plugin** — find its permission identifier in the plugin's documentation (e.g., `fs:allow-read`).
2. **Determine platform support** — check whether the plugin is available on mobile.
3. **Add to the correct file**:
   - Desktop-supported → add to `default.json` `permissions` array.
   - Mobile-supported → add to `mobile.json` `permissions` array as well.
   - Desktop-only → add only to `default.json`.
4. **Validate** — the `$schema` field enables IDE validation. Run `cargo tauri build` or `cargo tauri dev` to confirm no permission errors.

> **Warning:** Over-granting permissions weakens the security model. Always request the narrowest permission set (e.g., prefer `global-shortcut:allow-register` over `global-shortcut:allow-all` when only registration is needed).

## Relationship to the Rest of the Codebase

This module is **purely declarative** — it contains no executable code and defines no functions. It is consumed by the Tauri build system at compile time:

1. **Build time** — Tauri reads all files under `capabilities/`, resolves the permission tree, and generates a capability snapshot embedded in the binary.
2. **Runtime** — when frontend code or a Rust backend handler calls a plugin API, Tauri checks the snapshot before allowing the call. Unlisted permissions are rejected with a capability error.

Plugin initialization code in the application (Rust-side `Builder::plugin(...)` calls and frontend JavaScript imports) will silently fail or error if the corresponding capability permission is missing. Keep permissions in sync with the actual plugins registered in the app builder.