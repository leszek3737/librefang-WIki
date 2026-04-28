# Other — librefang-api-static

# librefang-api-static — Locale & Static Assets

## Purpose

This module holds static assets served by the LibreFang API, currently consisting of **i18n locale files** that provide all user-facing strings for the web dashboard. Every label, button, toast message, placeholder, and status indicator in the UI draws its text from these JSON dictionaries.

## File Layout

```
librefang-api/static/
└── locales/
    ├── en.json    ← Primary / fallback language
    └── ja.json    ← Japanese translation
```

The API serves these files to the frontend, which selects the appropriate locale at runtime based on the user's language preference.

## Locale File Structure

Each locale file is a single flat-ish JSON object with **top-level namespaces** that map to pages, components, or logical groupings. The convention is:

```
namespace.subNamespace.key
```

### Primary Namespaces

| Namespace | Scope |
|---|---|
| `nav` | Sidebar navigation labels |
| `status` | Connection / health status strings |
| `btn` | Reusable button labels (save, delete, cancel, etc.) |
| `label` | Generic field labels (name, key, value, status) |
| `auth` | API-key authentication gate |
| `page` | Page titles used in headers |
| `health` | Health-check status labels |
| `stat` | Stat card headings (agents running, tokens used, etc.) |
| `card` | Dashboard card titles |
| `agents` | Agent list page |
| `agentsPage` | Agent creation/management page |
| `detail` | Agent detail panel (info, files, config tabs) |
| `agentChat` | Agent chat interface — the largest namespace |
| `agentChat2` | Chat overlay controls (focus mode, model switch) |
| `sessionsPage` | Session browser & agent memory editor |
| `approvals` | Execution approval queue |
| `logsPage` | Live log stream & audit trail viewer |
| `runtimePage` | Runtime info page |
| `settingsPage` | Full settings page (providers, models, tools, security, network, budget, proactive memory, migration) |
| `settingsPage2` | Supplementary settings controls |
| `workflowsPage` | Workflow list & runner |
| `workflowBuilder` | Visual workflow builder (drag-and-drop canvas) |
| `schedulerPage` | Cron jobs, event triggers, run history |
| `channelsPage` | Messaging channel setup (Telegram, Discord, Slack, WhatsApp, etc.) |
| `skillsPage` | Skills browser, ClawHub integration, MCP servers, quick-start skills |
| `handsPage` | Hands — curated autonomous capability packages |
| `pluginsPage` | Plugin management |
| `commsPage` | Inter-agent communication & task board |
| `goalsPage` | Goal tracking with sub-goals |
| `analyticsPage` | Usage analytics, token/cost breakdowns |
| `memoryPage` | Proactive memory browser |
| `setupWizard` | First-run setup wizard (welcome → provider → agent → try → channel → done) |
| `security` | Security system short labels |
| `onboarding` | First-visit onboarding banner |
| `provider` | LLM provider setup |
| `overview` | Dashboard overview cards & recent activity |
| `time` | Relative time formatting |
| `mode` | Agent mode labels (Observe / Assist / Full) |
| `category` | Skill/template categories |
| `profile` | Tool profile descriptions (Minimal → Full) |
| `template` | Agent template names & descriptions |
| `theme` | Theme selector labels |
| `sidebar` | Sidebar keyboard shortcut hints |
| `confirm` | Generic confirm dialog buttons |
| `setupWizard2` | Supplementary wizard controls |

## Interpolation

Strings use **brace-delimited placeholders** for dynamic values:

```json
"stepsCompleted": "{count} of 5 steps completed"
"providerConnected": "{provider} connected ({latency}ms)"
```

The frontend substitutes these at render time. There is no pluralization library embedded in the locale files — plural handling is done per-key (e.g., separate `noSessionsYet` vs. `sessionsCreatedWhen`).

## Adding a New Language

1. Copy `en.json` to a new file named with the appropriate [ISO 639-1 code](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes) (e.g., `de.json` for German).
2. Translate all string values. **Do not change keys or structure** — the frontend references keys by path.
3. Keep interpolation tokens (`{count}`, `{message}`, `{name}`, etc.) intact in translated strings.
4. Register the new locale in the frontend's i18n configuration (outside this module).

## Adding New Strings

When adding UI features:

1. **Find the right namespace.** If the feature lives on an existing page, add keys to that page's namespace (e.g., `agentsPage.newKey`). For cross-cutting strings, use a generic namespace (`btn`, `label`, `status`).
2. **Add the key to every locale file.** Missing keys fall back to `en.json`, but leaving other locales incomplete degrades the experience for those users.
3. **Use descriptive key names.** Prefer `deleteFailed` over `error2`. Group related keys under a sub-object when there are more than a handful.

## Namespace Design Patterns

The files follow several conventions worth noting when extending them:

### Toast messages

Actions that produce transient feedback place toast strings adjacent to the action, not in a central `toast` namespace:

```json
"agentsPage": {
  "agentSpawned": "Agent \"{name}\" spawned",
  "spawnFailed": "Spawn failed: {message}"
}
```

### Nested sub-namespaces

Large pages use nested objects for logical grouping:

```json
"agentChat": {
  "cmd": { "help": "...", "model": "..." },
  "toast": { "modelSwitched": "...", "switchFailed": "..." },
  "sys": { "compacting": "...", "compactDone": "..." }
}
```

### Enum-like sets

Status values, filter options, and categories are grouped even when there are only a few:

```json
"approvals": {
  "filter": { "all": "All", "pending": "Pending", "approved": "Approved", "rejected": "Rejected" },
  "status": { "pending": "Pending", "approved": "Approved", "rejected": "Rejected", "expired": "Expired" }
}
```

### Shared vs. page-specific keys

Common actions (save, delete, cancel) live in `btn.*`. Page-specific labels live in their own namespace even if the text overlaps — this keeps each page self-contained and simplifies independent translation.

## Relationship to the Rest of the Codebase

- **Frontend components** import from these files via the i18n layer and reference keys by dotted path (e.g., `t('agentsPage.spawnAgent')`).
- **The API server** serves the `static/` directory as-is — no build step or transformation occurs server-side.
- **Security feature descriptions** in `settingsPage.coreFeatures`, `settingsPage.configurableFeatures`, and `settingsPage.monitoringFeatures` are the canonical user-facing descriptions of the security subsystems implemented in the Rust kernel. Keep these in sync when security behavior changes.