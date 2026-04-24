# Other — librefang-desktop-capabilities

# librefang-desktop-capabilities

## Overview

This module defines the **default capability set** for the LibreFang desktop application using Tauri's permission system. It is a static configuration file (`default.json`) that declares which Tauri plugins and core APIs the application is authorized to use.

Tauri's capability system acts as an allowlist: if a permission is not listed here, the corresponding API calls will be rejected at runtime. This file gates access for the **main window** of the application.

## File Location

```
librefang-desktop/capabilities/default.json
```

## Schema

The file is validated against Tauri's official schema:

```
https://raw.githubusercontent.com/nicedoc/tauri/refs/heads/dev/crates/tauri-utils/schema.json
```

Editor tooling that supports `$schema` (VS Code, JetBrains IDEs) will provide autocompletion and validation against this schema.

## Structure

| Field | Value | Purpose |
|---|---|---|
| `identifier` | `"default"` | Names this capability set; referenced internally by Tauri's permission resolver |
| `description` | Human-readable string | Documentation only — not evaluated at runtime |
| `windows` | `["main"]` | Restricts these permissions to the window labeled `"main"` |
| `permissions` | Array of strings | The granted permission identifiers |

## Granted Permissions

### Core

| Permission | Effect |
|---|---|
| `core:default` | Standard Tauri core APIs — window management, event system, basic IPC |

### Shell

| Permission | Effect |
|---|---|
| `shell:default` | Default shell operations (e.g., opening URLs in the system browser) |

### Dialog

| Permission | Effect |
|---|---|
| `dialog:default` | Native file picker, message boxes, and confirm dialogs |

### Notification

| Permission | Effect |
|---|---|
| `notification:default` | System notification delivery |

### Global Shortcut

| Permission | Effect |
|---|---|
| `global-shortcut:allow-register` | Register global keyboard shortcuts that work even when the app is not focused |
| `global-shortcut:allow-unregister` | Remove previously registered shortcuts |
| `global-shortcut:allow-is-registered` | Query whether a shortcut is currently registered |

Note: Global shortcut permissions are granted individually rather than via the `default` set, indicating the app uses explicit register/unregister/query control rather than blanket access.

### Autostart

| Permission | Effect |
|---|---|
| `autostart:default` | Read and write the application's launch-at-login configuration |

### Updater

| Permission | Effect |
|---|---|
| `updater:default` | Check for and apply application updates |

## How It Connects to the Codebase

This file has no imports, function calls, or runtime logic. It is consumed at **build time** by Tauri's capability resolver:

1. Tauri's build plugin reads all JSON files under the `capabilities/` directory.
2. The declared permissions are compiled into the application binary.
3. At runtime, when frontend JavaScript or Rust backend code invokes a Tauri command (e.g., registering a global shortcut), the permission is checked against this compiled set.
4. Unauthorized calls return a permission-denied error.

Because there are no additional capability files, this single file is the **complete** permission surface for the application.

## Modifying Permissions

To add a new permission (for example, filesystem access):

1. Add the permission identifier to the `permissions` array.
2. Rebuild the application — changes to this file are not hot-reloaded.
3. Verify that the corresponding Tauri plugin is also registered in the app's Rust-side builder (typically in `main.rs` or `lib.rs`), since a declared permission without a registered plugin has no effect.

To scope permissions to additional windows, add the window label to the `windows` array, or create a separate capability file with a different `identifier`.

## Security Considerations

- This file grants permissions to the **main** window. If the application opens additional windows (e.g., a popout or secondary view), they will have **no permissions** unless explicitly added.
- `shell:default` allows opening external URLs. Validate any user-controlled input before passing it to shell APIs to prevent unintended resource access.
- `global-shortcut` permissions allow key registration system-wide. Conflicting shortcuts with other applications should be handled gracefully via error handling around `register` calls.