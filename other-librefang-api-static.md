# Other — librefang-api-static

# Static Locale Files — `librefang-api/static/locales/`

## Purpose

This directory holds the **i18n translation catalogs** for the LibreFang web dashboard. Every user-facing string — navigation labels, button text, error messages, toast notifications, section descriptions, and onboarding content — is defined here rather than hardcoded in React components. The frontend's i18n layer (loaded via `react/assets/i18n-*.js`) reads these JSON files at runtime and resolves keys to the user's selected locale.

## Files

| File | Language | Status |
|---|---|---|
| `locales/en.json` | English | Primary / source of truth |
| `locales/ja.json` | Japanese | In-progress translation |

The English catalog is the canonical source. Other locale files should mirror its key structure exactly, translating only the leaf values.

## Key Namespacing Convention

Keys are organized into **top-level namespaces** that map to UI sections or functional domains. The naming convention uses camelCase for namespaces and keys:

```
namespace.subkey.leafKey
```

### Major namespaces

| Namespace | Scope | Example keys |
|---|---|---|
| `nav` | Sidebar navigation | `chat`, `agents`, `settings`, `memory` |
| `btn` | Shared button labels | `send`, `save`, `delete`, `cancel` |
| `label` | Generic field labels | `status`, `provider`, `model` |
| `auth` | API key authentication overlay | `title`, `hint`, `placeholder` |
| `status` | Connection/state badges | `connecting`, `ready`, `disconnected` |
| `time` | Relative timestamps | `now`, `minutesAgo`, `hoursAgo` |
| `overview` | Dashboard overview page | `connectionError`, `recentActivity`, `quickActions` |
| `agentChat` | Chat interface | `generating`, `thinking`, `welcomeTips`, `cmd.*` |
| `agentsPage` | Agent management | `createAgent`, `spawnAgent`, `archetype.*` |
| `sessionsPage` | Session/memory browser | `conversationSessions`, `agentMemory` |
| `settingsPage` | Settings tabs | `llmProviders`, `coreFeatures.*`, `configurableFeatures.*` |
| `logsPage` | Live log + audit trail | `auditTitle`, `chainVerified`, `verifyChain` |
| `approvals` | Execution approval queue | `pendingCount`, `approve`, `reject` |
| `workflowsPage` | Workflow editor | `createWorkflow`, `mode.*` |
| `workflowBuilder` | Visual workflow canvas | `nodePalette`, `nodeType.*`, `exportToml` |
| `schedulerPage` | Cron jobs + event triggers | `cron.*`, `triggerType.*` |
| `channelsPage` | Messaging channel config | `step.*`, `status.*`, `category.*` |
| `skillsPage` | Skills & ClawHub browser | `clawHubTab`, `category.*`, `quickStartSkill.*` |
| `handsPage` | Autonomous capability packages | `activate`, `status.*`, `step.*` |
| `pluginsPage` | Plugin management | `installedTab`, `source*` |
| `commsPage` | Inter-agent messaging | `event.*`, `state.*` |
| `setupWizard` | First-run wizard | `welcomeStep` through `doneStep` |
| `analyticsPage` | Usage/cost dashboards | `tokenBreakdown`, `dailyCost` |
| `memoryPage` | Proactive memory viewer | `totalMemories`, `level` |
| `goalsPage` | Goal tracking | `statusPending`, `addSubGoal` |
| `security` | Security feature names | `merkleAudit`, `taintTracking` |
| `profile` | Agent tool profiles | `minimal`, `coding`, `full` |
| `template` | Agent templates (10 built-in) | `GeneralAssistant`, `CodeHelper`, … |
| `onboarding` | First-visit banner | `welcome`, `launchWizard` |
| `provider` | LLM provider setup | `bannerText`, `ollamaDetected` |
| `detail` | Agent detail panel | `toolFilters`, `fallbacks`, `clone` |
| `mode` | Agent modes | `observe`, `assist`, `full` |
| `category` | Agent categories | `general`, `development`, `research` |
| `theme` | Theme switcher | `light`, `dark`, `system` |
| `sidebar` | Sidebar hints | `shortcutHint` |
| `confirm` | Confirmation dialog | `cancel`, `confirm` |

Several namespaces have a `*Page2` variant (e.g. `agentsPage2`, `analyticsPage2`) that contains supplementary keys for secondary UI elements — these are typically loaded in extended or refactored views.

## Interpolation

String values use **brace-delimited placeholders** for dynamic content:

```json
"minutesAgo": "{count}m ago",
"agentsRunning": "{count} agent(s) running",
"providerConnected": "{provider} connected ({latency}ms)",
"stopAgentConfirm": "Stop agent \"{name}\"? The agent will be shut down."
```

The frontend i18n runtime substitutes these at render time. When adding keys, follow these rules:

- Use `{variableName}` for substitutions
- Keep placeholder names semantic: `{count}`, `{name}`, `{provider}`, `{model}`, `{message}`
- For pluralizable values, let the component logic handle plural/singular; the string itself should be neutral (e.g., `"{count} agent(s) running"`)

## Nested sub-namespaces

Some namespaces use deeper nesting to group related keys:

```
agentChat.cmd.help
agentChat.cmd.model
agentChat.toast.modelSwitched
agentChat.sys.thinkOnDesc
settingsPage.coreFeatures.path_traversal.name
settingsPage.coreFeatures.path_traversal.description
schedulerPage.cron.everyMinute
schedulerPage.triggerType.lifecycle
skillsPage.quickStartSkill.code-review-guide.name
```

These correspond to discrete UI subsections (command definitions, toast messages, system messages, security feature details).

## Relationship to the React Frontend

```mermaid
graph LR
    A[locales/en.json] --> B[i18n runtime]
    C[locales/ja.json] --> B
    B --> D[React components]
    D -->|t("overview.recentActivity")| B
    B -->|"Recent Activity"| D
```

The i18n runtime (initialized in `react/assets/i18n-*.js`) loads these JSON files and exposes a `t()` function to all React components. Tests reference the locale data via `react/assets/index-*.js` (the `find` calls in integration tests verify that rendered output contains expected translated strings).

The locale files are served as **static assets** from the `static/locales/` directory — the Rust API server (`librefang-api`) mounts them via a static file handler, making them available to the browser at runtime.

## Adding a New Locale

1. Copy `en.json` to a new file named with the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g. `de.json`, `fr.json`, `zh-CN.json`)
2. Translate all leaf string values — **do not** modify any keys
3. Leave any ICU-format placeholders (`{count}`, `{name}`) untouched in the translated strings
4. Register the new locale in the frontend i18n configuration (the `i18n-*.js` bundle)
5. Run the `dashboard/scripts/i18n-parity.mjs` script to verify the new file has parity with `en.json`

### Parity checking

The `i18n-parity.mjs` script validates that every key in `en.json` exists in all other locale files and vice versa. It flags:
- Missing keys (present in `en.json` but absent from the target locale)
- Extra keys (present in the target locale but not in `en.json`)

## Adding New Keys

When adding a new UI feature:

1. Choose the appropriate namespace. For a new page, create a new top-level namespace following the `*Page` convention.
2. Group related strings under sub-keys (e.g., `cmd.*` for commands, `toast.*` for notifications, `status.*` for state labels).
3. Add the key to **all** locale files. For non-English locales, either provide the translation or copy the English value as a placeholder with a `// TODO` comment in a parallel tracking issue.
4. Use the established naming patterns:
   - `*Title` for dialog/section headings
   - `*Desc` for description text
   - `*Placeholder` for input placeholders
   - `*Confirm` for confirmation dialog text
   - `*Failed` for error toast messages
   - `*Toast` for success toast messages

## Key Coverage Summary

The English catalog (`en.json`) contains approximately **850+ translation keys** spanning:

- **22 page-level namespaces** (overview, agents, sessions, settings, logs, approvals, workflows, scheduler, channels, skills, hands, plugins, comms, analytics, memory, goals, runtime, etc.)
- **10 agent templates** (GeneralAssistant, CodeHelper, Researcher, Writer, DataAnalyst, DevOpsEngineer, CustomerSupport, Tutor, APIDesigner, MeetingNotes)
- **9 tool profiles** (minimal through full)
- **8 security core features** with full descriptions
- **4 configurable security features** with descriptions and tuning hints
- **3 monitoring features**
- **18 slash commands** (help, new, reboot, compact, model, think, context, verbose, etc.)
- **20+ cron presets** (every minute through monthly schedules)
- **18 skill categories** and **6 channel categories**
- **4 quick-start skills**