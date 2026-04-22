# Other — librefang-desktop-gen

# librefang-desktop-gen

Auto-generated Tauri v2 security schemas that define the IPC permission boundary between the LibreFang desktop frontend and the native backend. This directory is produced by Tauri's build process and should not be edited directly.

## Directory Structure

```
gen/
└── schemas/
    ├── acl-manifests.json    # Plugin permission manifests
    ├── capabilities.json     # Resolved capability grants
    └── desktop-schema.json   # JSON Schema for capability files
```

## How It Works

Tauri v2 uses a capability-based security model. The webview frontend has zero access to native APIs unless a capability explicitly grants it. At build time, Tauri:

1. Reads each plugin's permission definitions from `acl-manifests.json`
2. Resolves the capabilities declared in the project's `capabilities/` directory
3. Writes the resolved result to `capabilities.json`
4. Generates `desktop-schema.json` for IDE validation of capability files

The frontend JavaScript bindings (`@tauri-apps/api`, `@tauri-apps/plugin-*`) check these permissions at runtime. Calling a command without its matching permission grant results in a runtime error.

## Default Capability

The application defines a single capability (`capabilities.json`) applied to the `"main"` window:

```json
{
  "identifier": "default",
  "local": true,
  "windows": ["main"],
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

### Granted Permission Summary

| Permission Set | What It Allows |
|---|---|
| `core:default` | Path, event, window, webview, app, image, resources, menu, and tray operations (read-heavy defaults) |
| `notification:default` | Full notification lifecycle: permission requests, sending, channels, batching, cancellation |
| `shell:default` | Opening `http(s)://`, `tel:`, and `mailto:` links via the system handler |
| `dialog:default` | All native dialog types: `ask`, `confirm`, `message`, `save`, `open` |
| `global-shortcut:allow-*` | Registering, unregistering, and checking global keyboard shortcuts (no `register-all` or `unregister-all`) |
| `autostart:default` | Enabling, disabling, and querying OS login boot state |
| `updater:default` | Full update lifecycle: `check`, `download`, `install`, `download_and_install` |

## Plugin Permission Reference

Each plugin in `acl-manifests.json` follows the same structure:

```json
{
  "plugin-name": {
    "default_permission": { "identifier": "default", "permissions": ["..."] },
    "permissions": {
      "allow-command": { "commands": { "allow": ["command"], "deny": [] } },
      "deny-command":  { "commands": { "allow": [], "deny": ["command"] } }
    },
    "permission_sets": {},
    "global_scope_schema": null
  }
}
```

### Permission Naming Convention

- `allow-{command}` — whitelists a specific IPC command
- `deny-{command}` — explicitly blocks a command (takes priority over allow)
- `{plugin}:default` — bundles a curated set of `allow-*` permissions

### Plugins Included

| Plugin | Default Commands (selected) |
|---|---|
| **autostart** | `enable`, `disable`, `is_enabled` |
| **core:app** | `version`, `name`, `tauri_version`, `identifier`, `bundle_type`, `register_listener`, `remove_listener` |
| **core:event** | `listen`, `unlisten`, `emit`, `emit_to` |
| **core:image** | `new`, `from_bytes`, `from_path`, `rgba`, `size` |
| **core:menu** | `new`, `append`, `prepend`, `insert`, `remove`, `popup`, `set_as_app_menu`, `set_text`, `set_accelerator`, etc. |
| **core:path** | `resolve`, `resolve_directory`, `join`, `dirname`, `basename`, `extname`, `normalize`, `is_absolute` |
| **core:resources** | `close` |
| **core:tray** | `new`, `get_by_id`, `remove_by_id`, `set_icon`, `set_menu`, `set_tooltip`, `set_title`, `set_visible` |
| **core:webview** | `get_all_webviews`, `webview_position`, `webview_size`, `internal_toggle_devtools` (default only) |
| **core:window** | Position/size/state queries (default); mutation commands like `maximize`, `minimize`, `close`, `set_size`, `set_position` available but not in default |
| **dialog** | `ask`, `confirm`, `message`, `save`, `open` |
| **global-shortcut** | No default (empty); individual `register`/`unregister`/`is_registered` granted in the capability |
| **notification** | Full lifecycle: `notify`, `request_permission`, `is_permission_granted`, `register_action_types`, `cancel`, `get_pending`, `get_active`, `show`, `batch`, channel management |
| **shell** | `open` only by default; `execute`, `spawn`, `kill`, `stdin_write` available but not granted |
| **updater** | `check`, `download`, `install`, `download_and_install` |

## Shell Scope Schema

The `shell` plugin is the only plugin with a `global_scope_schema`, which validates shell command configurations. Scope entries support two forms:

**System command:**
```json
{ "name": "my-cmd", "cmd": "$HOME/script.sh", "args": true }
```

**Sidecar:**
```json
{ "name": "my-sidecar", "sidecar": true, "args": [{ "validator": "\\w+" }] }
```

Arguments can be:
- `true` — allow any arguments
- `false` — deny all arguments
- An array of strings (fixed args) or objects with a `validator` regex

Path variables available in `cmd`: `$AUDIO`, `$CACHE`, `$CONFIG`, `$DATA`, `$LOCALDATA`, `$DESKTOP`, `$DOCUMENT`, `$DOWNLOAD`, `$EXE`, `$FONT`, `$HOME`, `$PICTURE`, `$PUBLIC`, `$RUNTIME`, `$TEMPLATE`, `$VIDEO`, `$RESOURCE`, `$LOG`, `$TEMP`, `$APPCONFIG`, `$APPDATA`, `$APPLOCALDATA`, `$APPCACHE`, `$APPLOG`.

## Capability File Format

`desktop-schema.json` defines the valid structure for capability files used in the project's `src-tauri/capabilities/` directory. A capability file can be:

- A single `Capability` object
- An array of `Capability` objects
- An object with a `capabilities` array property

Each `Capability` has:

| Field | Required | Description |
|---|---|---|
| `identifier` | Yes | Unique name for the capability |
| `permissions` | Yes | Array of permission identifiers or scoped permission objects |
| `description` | No | Human-readable purpose |
| `windows` | No | Window labels this applies to (supports glob patterns) |
| `webviews` | No | Webview labels this applies to (finer-grained than windows) |
| `local` | No | Enable for local app URLs (default: `true`) |
| `remote` | No | Remote URL patterns allowed to use these permissions |
| `platforms` | No | Restrict to specific OS targets |

## Modifying Permissions

To change the app's permission grants:

1. Edit files in `src-tauri/capabilities/` (not this `gen/` directory)
2. Rebuild the project — Tauri regenerates `gen/schemas/` automatically
3. The regenerated files reflect the updated capability configuration

Never edit files in `gen/schemas/` directly. They are overwritten on every build.