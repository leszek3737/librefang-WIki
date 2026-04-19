# Other — librefang-api-static

# librefang-api-static — Locale / i18n Translations

## Purpose

This directory holds the **static internationalization (i18n) translation files** for the LibreFang dashboard UI. Every piece of user-facing text in the web frontend — button labels, page titles, error messages, toast notifications, section descriptions, and contextual help — is defined here rather than hardcoded in components. This allows the UI to be rendered in multiple languages by swapping locale files at runtime.

## Files

| File | Language | Status |
|---|---|---|
| `locales/en.json` | English | Complete — authoritative source |
| `locales/ja.json` | Japanese | Complete (parallel structure to `en.json`) |

Additional locales can be added by creating a new `{lang_code}.json` file that mirrors the same key structure.

## Structure and Key Organization

Translation keys are organized by **UI section or feature page**, using dot-notation paths that correspond to the logical grouping of components. The top-level namespaces are:

```
nav            – Sidebar / top navigation labels
status         – Connection and system status strings
btn            – Shared button labels (save, delete, cancel, etc.)
label          – Common field labels
auth           – API key authentication prompt
page           – Page-level headings
health         – Health check status
stat           – Dashboard stat card labels
card           – Dashboard card titles
agents         – Agent list page
detail         – Agent detail panel (info, files, config tabs)
mode           – Agent mode labels (observe, assist, full)
category       – Template category filters
profile        – Tool profile labels and descriptions
template       – Agent template names and descriptions
time           – Relative time formatting
onboarding     – First-run onboarding banner
provider       – LLM provider setup strings
overview       – Overview / dashboard page
security       – Security feature short names
agentChat      – Chat interface (messages, commands, toasts)
sessionsPage   – Sessions management page
agentsPage     – Agent creation and management page
approvals      – Execution approval queue
logsPage       – Live logs and audit trail
runtimePage    – Runtime information page
settingsPage   – Settings (providers, models, tools, security, network, budget, memory, migration)
workflowsPage  – Workflow creation and execution
workflowBuilder – Visual workflow builder (node palette, canvas controls)
schedulerPage  – Cron jobs, event triggers, run history
channelsPage   – Messaging channel configuration
skillsPage     – Skills marketplace, MCP servers, quick-start skills
handsPage      – Curated autonomous capability packages
pluginsPage    – Plugin management
commsPage      – Inter-agent communication
setupWizard    – Guided setup wizard (5 steps)
goalsPage      – Goal tracking
analyticsPage  – Usage analytics, costs, breakdowns
memoryPage     – Proactive memory browser
theme          – Theme selector labels
sidebar        – Sidebar shortcut hints
confirm        – Shared confirmation dialog buttons
```

Additional supplemental namespaces (`agentChat2`, `settingsPage2`, `agentsPage2`, etc.) hold secondary/extended keys that are added incrementally without modifying the primary namespace.

## Interpolation Syntax

Strings use **brace-delimited placeholders** for dynamic values:

```json
"secondsAgo": "{count}s ago",
"providersConfigured": "{configured}/{total} configured",
"errorPrefix": "Error: {message}"
```

The frontend rendering layer substitutes these at runtime with actual values. There is no pluralization library in the JSON itself — plural handling is done either by the consuming component or by having separate keys for singular/plural forms.

## Adding a New Locale

1. Copy `locales/en.json` to `locales/{lang_code}.json`.
2. Translate all string values while **preserving the exact key paths**. Do not rename, remove, or reorder keys.
3. Placeholders like `{count}`, `{name}`, `{message}` must remain untranslated — they are substitution tokens.
4. Register the new locale in the frontend's i18n initialization code (typically wherever the locale loader/importer is configured).

**Checklist for completeness:**

- Every top-level namespace present in `en.json` exists in the new file.
- All nested keys within each namespace match.
- All interpolation placeholders are preserved verbatim.
- No keys are missing — missing keys will fall back to English or render the raw key path.

## Adding New UI Strings

When new UI features are added:

1. Determine the appropriate top-level namespace. If the feature is a new page, create a new namespace (e.g., `newFeaturePage`). If it extends an existing page, add keys under the existing namespace.
2. Add the key to **`en.json` first** — it is the canonical source.
3. Add the same key (translated) to all other locale files.
4. Use the key in the frontend component via the i18n library's lookup function (e.g., `t('newFeaturePage.title')`).

## Conventions

| Convention | Example |
|---|---|
| Toast notifications use past tense or result state | `"agentStopped": "Agent \"{name}\" stopped"` |
| Error messages include the dynamic `{message}` placeholder | `"loadFailed": "Failed: {message}"` |
| Confirmation dialogs have paired `Title`/`Confirm` keys | `"stopAgentTitle"` / `"stopAgentConfirm"` |
| Status enums use lowercase or capitalized single words | `"pending"`, `"Approved"` |
| Button labels are imperative verbs | `"save"`, `"delete"`, `"cancel"` |
| Section descriptions end with periods | `"...to enrich context."` |
| Nested enums go under a sub-object | `"filter"`, `"status"`, `"cmd"`, `"toast"`, `"sys"` |

## Relationship to the Rest of the Codebase

```
┌─────────────────────┐
│  Frontend Components │  ← Reference keys via t('namespace.key')
│  (Svelte / React)   │
└────────┬────────────┘
         │ imports
┌────────▼────────────┐
│  i18n Runtime Layer  │  ← Loads active locale, resolves keys
└────────┬────────────┘
         │ reads
┌────────▼────────────┐
│  locales/*.json      │  ← This module
└─────────────────────┘
```

The locale files are **pure data** — no code, no imports, no side effects. They are consumed at build time or runtime by the frontend's i18n system. The backend API serves them as static assets under the `/static/locales/` path.

Because the files contain no executable logic, there are no call graph edges or execution flows associated with this module.