# Other — librefang-desktop-gen

# librefang-desktop/gen

Auto-generated Tauri build configuration directory. This directory is populated by the Tauri CLI and governs the IPC security model, mobile scaffolding, and capability schemas for the LibreFang desktop application.

**Do not hand-edit generated files.** Changes will be overwritten on the next `tauri build` or `tauri dev` invocation.

## Directory Layout

```
gen/
├── android/                # Android project scaffold (via cargo tauri android init)
│   └── README.md
├── apple/                  # iOS/macOS project scaffold (via cargo tauri ios init)
│   └── README.md
└── schemas/
    ├── acl-manifests.json  # Permission manifests for every Tauri plugin
    ├── capabilities.json   # Active capability sets for this application
    └── desktop-schema.json # JSON Schema validating capability files
```

## Mobile Scaffolding

The `android/` and `apple/` directories are placeholders. They are populated by running:

```bash
# From crates/librefang-desktop/
cargo tauri android init   # Android (all platforms)
cargo tauri ios init       # iOS/macOS (macOS host only)
```

These directories hold the native wrapper projects that Tauri uses to build mobile bundles. Until the init commands are run, each directory only contains a README stub.

## Capabilities Configuration

The active security policy is defined in `schemas/capabilities.json`. Two capability sets exist, scoped by target platform.

### Desktop Capability (`default`)

Applies to the `main` window on **macOS, Windows, and Linux**.

| Permission | Scope |
|---|---|
| `core:default` | All core plugins (path, event, window, webview, app, image, resources, menu, tray) |
| `notification:default` | Full notification workflow (request permission, show, batch, channels, etc.) |
| `shell:default` | `open` for `http(s)://`, `tel:`, `mailto:` URLs |
| `dialog:default` | Message, save, and open dialogs |
| `global-shortcut:allow-register` | Register keyboard shortcuts |
| `global-shortcut:allow-unregister` | Unregister keyboard shortcuts |
| `global-shortcut:allow-is-registered` | Check if a shortcut is registered |
| `autostart:default` | Enable, disable, and query auto-start on boot |
| `updater:default` | Check, download, and install application updates |

### Mobile Capability (`mobile`)

Applies to the `main` window on **iOS and Android**. Desktop-only plugins (shell, global-shortcut, autostart, updater) are excluded.

| Permission | Scope |
|---|---|
| `core:default` | All core plugins |
| `notification:default` | Full notification workflow |
| `dialog:default` | Message, save, and open dialogs |

## ACL Manifests

`schemas/acl-manifests.json` declares every permission available from each bundled Tauri plugin. Each entry contains:

- **`default_permission`** — the permission set applied when referencing `<plugin>:default`
- **`permissions`** — individual `allow-*` and `deny-*` entries mapping to specific IPC commands
- **`global_scope_schema`** — optional JSON Schema constraining scope data (e.g., `shell` defines allowed commands and arguments)

### Plugins in Use

| Plugin | Default Commands Enabled |
|---|---|
| `autostart` | `enable`, `disable`, `is_enabled` |
| `core:app` | `version`, `name`, `tauri_version`, `identifier`, `bundle_type`, `register_listener`, `remove_listener`, `supports_multiple_windows` |
| `core:event` | `listen`, `unlisten`, `emit`, `emit_to` |
| `core:image` | `new`, `from_bytes`, `from_path`, `rgba`, `size` |
| `core:menu` | Full menu CRUD (`new`, `append`, `prepend`, `insert`, `remove`, `popup`, `set_as_app_menu`, `set_text`, `set_enabled`, `set_checked`, etc.) |
| `core:path` | `resolve`, `resolve_directory`, `normalize`, `join`, `dirname`, `extname`, `basename`, `is_absolute` |
| `core:resources` | `close` |
| `core:tray` | `new`, `get_by_id`, `remove_by_id`, `set_icon`, `set_menu`, `set_tooltip`, `set_title`, `set_visible`, etc. |
| `core:webview` | `get_all_webviews`, `webview_position`, `webview_size`, `internal_toggle_devtools` |
| `core:window` | Extensive — all read-only window queries (`is_fullscreen`, `is_minimized`, `inner_size`, `theme`, etc.) plus `internal_toggle_maximize` |
| `dialog` | `message`, `save`, `open` |
| `global-shortcut` | None by default; explicitly granted per-command |
| `notification` | Full workflow: permission checks, show, batch, channels, cancel, listeners |
| `shell` | `open` only (with URL scope validation) |
| `updater` | `check`, `download`, `install`, `download_and_install` |

## Desktop Schema

`schemas/desktop-schema.json` is a JSON Schema (draft-07) defining the structure of `CapabilityFile` documents. It validates:

- **`Capability`** objects with fields: `identifier`, `permissions`, `windows`, `webviews`, `platforms`, `local`, `remote`
- **`PermissionEntry`** — either a plain identifier string or an object with `identifier` + scoped `allow`/`deny` arrays
- **Shell scope entries** with support for `cmd` paths (using `$HOME`, `$APPDATA`, etc. variables), sidecar commands, and argument validators (regex-based)

This schema is used by `tauri build` to validate capability files before code generation.

## Regenerating

After modifying permissions in the Tauri configuration or adding/removing plugins, regenerate from the crate root:

```bash
cd crates/librefang-desktop
cargo tauri dev      # regenerates schemas and rebuilds
```