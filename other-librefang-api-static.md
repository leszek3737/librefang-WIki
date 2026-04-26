# Other — librefang-api-static

# librefang-api-static — UI Locale Files

## Purpose

This module holds the **static internationalization (i18n) locale files** for the LibreFang web dashboard. Every user-facing string rendered in the browser — button labels, status messages, toast notifications, page headings, error text, and onboarding copy — is defined here rather than hard-coded in components.

Two locales are currently provided:

| File | Language | Role |
|---|---|---|
| `locales/en.json` | English (`en`) | Primary / fallback locale |
| `locales/ja.json` | Japanese (`ja`) | Secondary locale |

The frontend i18n library loads the matching JSON at runtime based on the user's language preference and falls back to `en.json` when a key is missing from the active locale.

## Structure of a Locale File

Each JSON file is a single flat-to-nested object. Keys are grouped by **logical feature area** (page, component, or domain concept). The top-level namespaces are:

```
nav          – sidebar / top-bar navigation labels
status       – connection / readiness status strings
btn          – reusable button labels
label        – generic field labels
auth         – API-key gate screen
page         – page-level titles
health       – health-check badges
stat         – dashboard stat card headers
card         – dashboard card titles
agents       – agent list header actions
detail       – agent detail pane (Info / Files / Config tabs)
mode         – observe / assist / full mode labels
category     – agent template categories
profile      – tool profile names & descriptions
template     – 10 built-in agent templates (name + desc)
time         – relative-time formatting
onboarding   – first-launch banner
provider     – LLM provider setup banner
overview     – Overview dashboard page
security     – security system short names
agentChat    – chat view (messages, commands, toasts, sys messages)
sessionsPage – Sessions & Memory management page
agentsPage   – Agents page (create, spawn, stop, config)
approvals    – execution approval queue
logsPage     – live logs + audit trail
runtimePage  – runtime info display
settingsPage – Settings (providers, models, tools, security, network,
               budget, proactive memory, system, migration)
workflowsPage – workflow list & execution
workflowBuilder – visual workflow builder (node palette, canvas)
schedulerPage – cron jobs, event triggers, run history
channelsPage – messaging channel setup (Telegram, Discord, Slack, etc.)
skillsPage   – skills browser, ClawHub, MCP servers
handsPage    – curated autonomous capability packages
pluginsPage  – plugin management
commsPage    – inter-agent communication / topology
setupWizard  – guided first-run wizard (5 steps)
goalsPage    – hierarchical goals with sub-goals
analyticsPage – token/cost analytics (summary, by model, by agent, costs)
memoryPage   – proactive memory browser
theme        – light / dark / system toggle
sidebar      – keyboard shortcut hints
confirm      – generic confirm dialog buttons
```

Supplementary namespaces (`agentChat2`, `settingsPage2`, `agentsPage2`, etc.) hold additional keys that are appended or overridden for specific UI variants.

## Interpolation

Strings use `{placeholder}` syntax for dynamic values. The frontend i18n layer replaces these at render time.

Examples from `en.json`:

```json
"stepsCompleted": "{count} of 5 steps completed",
"agentsStopped": "{count} agent(s) stopped",
"providerConnected": "{provider} connected ({latency}ms)",
"keySaved": "Key \"{key}\" saved"
```

All placeholders must be preserved exactly when translating. A missing placeholder in a translated string will surface as the literal text `{placeholderName}` in the UI.

## Adding a New String

1. **Identify the correct namespace.** If the string belongs to an existing page, add it under that page's object (e.g. `settingsPage.budgetTitle`). For cross-cutting UI elements, use `btn`, `label`, or `status`.

2. **Add the key to `en.json` first.** This is the canonical source of truth.

3. **Add the translated key to every other locale file** (`ja.json`, and any future locales). If the translation is not yet available, copy the English value — the i18n fallback mechanism will handle it, but explicit entries make it obvious what still needs translation.

4. **Use the key in the component** via whatever i18n function the frontend provides (e.g. `t('settingsPage.budgetTitle')`).

## Adding a New Locale

1. Create `locales/<code>.json` (e.g. `locales/fr.json`).
2. Copy the full structure from `en.json` — every key must exist.
3. Translate all values, keeping `{placeholder}` tokens intact.
4. Register the new locale in the frontend i18n configuration so it is loaded and selectable in the theme/language picker.

## Key Conventions

| Convention | Example | Purpose |
|---|---|---|
| **Page-scoped keys** | `logsPage.title`, `runtimePage.agents` | Groups strings with the page that renders them |
| **Nested sub-groups** | `logsPage.filter.pending`, `agentChat.cmd.help` | Avoids long flat key names; mirrors component hierarchy |
| **Toast messages** | `agentsPage.agentStopped` | User-visible success/error notifications |
| **Error messages** | `agentsPage.spawnFailed` | Always include `{message}` placeholder for backend error detail |
| **Confirm dialogs** | `agentsPage.stopAgentTitle`, `agentsPage.stopAgentConfirm` | Paired title + body text for destructive action modals |
| **Status enums** | `approvals.status.pending`, `commsPage.state.running` | Finite-state display labels |

## Relationship to the Rest of the Codebase

```mermaid
graph LR
    A[Frontend Components] -->|t('key')| B[i18n Library]
    B -->|loads| C[locales/en.json]
    B -->|loads| D[locales/ja.json]
    A -->|renders strings| E[Browser DOM]
    F[Backend API] -->|error messages| A
```

- **Frontend components** call `t('namespace.key')` to resolve a localized string.
- The **i18n library** loads the active locale JSON at app init and falls back to `en` for missing keys.
- **Backend API responses** return raw error text; the frontend wraps it via `{message}` placeholders (e.g. `spawnFailed: "Spawn failed: {message}"`).
- No backend code reads these files — they are served purely as static assets to the browser.

## Coverage Notes

The English locale (`en.json`) contains approximately **1 100+ keys** covering 25+ page/feature areas. The Japanese locale (`ja.json`) provides full parallel coverage. The `ja.json` file is truncated in the source snapshot but follows the identical structure; any missing tail keys fall back to English automatically.