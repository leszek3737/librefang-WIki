# Other — librefang-desktop-capabilities

# LibreFang Desktop — Capabilities (`default.json`)

## Purpose

This module defines the **Tauri capability set** for the LibreFang desktop application. Tauri uses a capability-based security model: every privileged operation (file dialogs, notifications, global shortcuts, etc.) must be explicitly declared before the frontend can invoke it. `default.json` is the single capability manifest applied to the main application window at runtime.

## Location

```
librefang-desktop/capabilities/default.json
```

## How Tauri Capabilities Work

When the app starts, the Tauri runtime reads every JSON file under `capabilities/` and builds an allow-list of IPC commands the webview is permitted to call. Each entry in the `permissions` array maps to one or more plugin commands exposed by the backend (Rust) side. If a permission is missing, any frontend call to that command will be rejected at the IPC boundary.

## Permission Reference

| Permission | Plugin | What It Enables |
|---|---|---|
| `core:default` | `tauri` | Core Tauri IPC, window management, event system, and other baseline operations. |
| `notification:default` | `tauri-plugin-notification` | Sending native desktop notifications. |
| `shell:default` | `tauri-plugin-shell` | Spawning child processes and executing shell commands from the frontend. |
| `dialog:default` | `tauri-plugin-dialog` | Native file open/save pickers, message boxes, and input dialogs. |
| `global-shortcut:allow-register` | `tauri-plugin-global-shortcut` | Registering a system-wide keyboard shortcut. |
| `global-shortcut:allow-unregister` | `tauri-plugin-global-shortcut` | Removing a previously registered shortcut. |
| `global-shortcut:allow-is-registered` | `tauri-plugin-global-shortcut` | Querying whether a shortcut is currently registered. |
| `autostart:default` | `tauri-plugin-autostart` | Launching the app automatically when the user logs in. |
| `updater:default` | `tauri-plugin-updater` | Checking for, downloading, and applying application updates. |

### Granular vs Default Permissions

Most plugins offer a `default` shorthand that bundles common commands. The `global-shortcut` plugin is the exception here — instead of `global-shortcut:default`, the config explicitly lists the three individual commands (`allow-register`, `allow-unregister`, `allow-is-registered`). This follows the **principle of least privilege**: only the specific shortcut operations the app needs are exposed, rather than enabling every command the plugin provides.

## Schema Validation

The file references the official Tauri JSON schema:

```
"$schema": "https://raw.githubusercontent.com/nicedoc/tauri/refs/heads/dev/crates/tauri-utils/schema.json"
```

IDEs that support JSON Schema (VS Code, JetBrains, etc.) will use this for autocomplete and validation while editing.

## Scope

```json
"windows": ["main"]
```

All permissions are scoped exclusively to the `main` window. If additional windows are created at runtime, they will not inherit these capabilities unless explicitly added to this array or defined in a separate capability file.

## Adding a New Permission

When introducing a new Tauri plugin to the project:

1. Add the plugin crate to `Cargo.toml` and register it on the Tauri builder in the Rust entry point.
2. Add the corresponding permission identifier to the `permissions` array in `default.json`.
3. If the plugin provides fine-grained commands, prefer listing only the specific `allow-*` entries you need rather than using the `default` blanket.

## Relationship to the Rest of the Codebase

This file has no import/export graph — it is consumed directly by the Tauri runtime during app initialization. It does, however, serve as the **single gatekeeper** that determines which Tauri IPC commands the frontend JavaScript/TypeScript code can successfully invoke. Any feature in the UI layer that calls `invoke("plugin:command")` depends on a matching entry here.