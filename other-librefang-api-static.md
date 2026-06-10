# Other — librefang-api-static

# librefang-api-static — Locale & Static Asset Documentation

## Overview

The `librefang-api/static/` directory serves the LibreFang dashboard's static assets, with its primary contents being **i18n locale files** under `locales/`. These JSON files drive the entire dashboard's internationalized UI text — every label, button, tooltip, toast message, error string, and placeholder visible in the frontend is defined here, keyed by feature domain.

The dashboard consumes these files via an i18next-based internationalization layer (referenced in the bundled `i18n-4qyNqnlA.js`). The locale files are loaded at runtime and selected based on the user's browser language or explicit preference.

## Supported Languages

| File | Language | Code |
|------|----------|------|
| `locales/en.json` | English (default) | `en` |
| `locales/ja.json` | Japanese | `ja` |

English is the canonical/fallback language. All keys must exist in `en.json`; other languages may omit keys and fall back to English via i18next's resolution chain.

## Locale File Structure

Each locale file is a flat-nested JSON object. Top-level keys represent **feature domains** (pages, shared UI elements, or subsystems). Values are either strings or recursively nested objects for sub-grouping.

```
{
  "domain": {
    "subDomain": {
      "key": "Translated string with {variable}"
    }
  }
}
```

### Top-Level Domains

| Domain | Purpose |
|--------|---------|
| `nav` | Sidebar and top-level navigation labels |
| `status` | Connection and system status strings |
| `btn` | Shared button labels used across all pages |
| `label` | Generic form/field labels |
| `auth` | API key authentication prompt |
| `page` | Page title overrides |
| `health` | Health check status labels |
| `stat` | Dashboard stat card labels |
| `card` | Dashboard card titles |
| `agents` | Agent list page shared strings |
| `detail` | Agent detail panel labels |
| `mode` | Agent execution modes (Observe/Assist/Full) |
| `category` | Agent category filters |
| `profile` | Tool profile descriptions (Minimal through Full) |
| `template` | Agent template names and descriptions |
| `time` | Relative time formatting |
| `onboarding` | First-run onboarding messages |
| `provider` | LLM provider configuration strings |
| `overview` | Overview/dashboard page |
| `security` | Security feature short names |
| `agentChat` | Chat interface — the largest domain |
| `sessionsPage` | Sessions management page |
| `agentsPage` | Agent creation/management page |
| `approvals` | Execution approval workflow |
| `logsPage` | Live logs and audit trail |
| `runtimePage` | Runtime information page |
| `settingsPage` | Full settings page (providers, models, tools, security, network, budget, migration) |
| `workflowsPage` | Workflow list and execution |
| `workflowBuilder` | Visual workflow builder (drag-and-drop canvas) |
| `schedulerPage` | Cron scheduler and event triggers |
| `channelsPage` | Messaging channel configuration |
| `skillsPage` | Skills marketplace and MCP servers |
| `handsPage` | Autonomous capability packages |
| `pluginsPage` | Plugin management |
| `commsPage` | Inter-agent communication |
| `setupWizard` | Guided setup wizard (5-step flow) |
| `goalsPage` | Goal tracking |
| `analyticsPage` | Usage analytics and cost tracking |
| `memoryPage` | Proactive memory browser |
| `theme` | Theme toggle labels |
| `sidebar` | Sidebar hints |
| `confirm` | Generic confirm dialog |
| `agentChat2`, `settingsPage2`, `agentsPage2`, `schedulerPage2`, `analyticsPage2`, `memoryPage2`, `setupWizard2` | Supplementary keys for extended/alternate UI variants |

## Interpolation Format

Translation strings use **i18next interpolation syntax**:

```
"{count} agent(s) running"
"Failed: {message}"
"Switched to {model}"
```

- Variables are wrapped in `{` `}` (single curly braces).
- The frontend passes a context object when calling `t()`:
  ```js
  t("stat.agentsRunning", { count: 3 })
  // → "3 agent(s) running"
  ```

### Pluralization

Pluralization is handled inline (e.g., `"{count} agent(s) running"`) rather than through i18next's dedicated plural suffixes (`_one`, `_other`). This is a deliberate simplification — the English source uses parenthetical plurals, and localized versions adapt as needed.

## Key Sections in Detail

### `agentChat` — Chat Interface

The largest domain (~200 keys), covering:
- Message display (`generating`, `thinking`, `usingTool`, `working`)
- Session management (`sessions`, `newSession`, `switchSessionFailed`)
- Slash commands (`cmd.help`, `cmd.model`, `cmd.think`, etc.)
- Toast notifications (`toast.modelSwitched`, `toast.sessionReset`)
- System messages (`sys.compacting`, `sys.budgetStatus`, `sys.ofpNetwork`)
- File handling (`dropFilesHere`, `fileTooLarge`, `failedUploadFile`)
- Keyboard shortcuts and tips (`welcomeTips`, `tipCommands`, `tipFocus`)

### `settingsPage` — Settings

The second-largest domain, organized into sub-domains matching the settings tabs:

- **LLM Providers** — `llmProviders`, `addCustomProvider`, per-provider connection strings
- **Model Catalog** — `modelCatalog`, `catalogSync`, tier/filter labels
- **Tools** — `toolsTab`, search and filter strings
- **Security** — Three sub-groups:
  - `coreFeatures` — 8 always-on protections (path traversal, SSRF, capability system, etc.) with `name`, `description`, and `threat` fields
  - `configurableFeatures` — 4 tunable controls (rate limiter, WebSocket limits, WASM sandbox, auth) with `name`, `description`, `hint`
  - `monitoringFeatures` — 3 monitoring systems (audit trail, taint tracking, manifest signing)
- **Network** — OFP peer networking, A2A external agents
- **Budget** — Spending limits, cost tracking, alert thresholds
- **Proactive Memory** — mem0-style memory configuration (`autoMemorize`, `confidenceDecayRate`, `duplicateThreshold`, etc.)
- **Migration** — OpenClaw/OpenFang data import
- **Security Dependencies** — `securityDependency` sub-keys for cryptographic primitives

### `profile` — Tool Profiles

Nine tool profiles define agent capability sets:

```json
"profile": {
  "minimal": { "label": "Minimal", "desc": "Read-only file access" },
  "coding":   { "label": "Coding",  "desc": "Files + shell + web fetch" },
  // ... through "full": "All 35+ tools"
}
```

Each profile has a `label` (display name) and `desc` (short description shown in the UI).

### `template` — Agent Templates

Ten built-in agent templates:

| Key | Template |
|-----|----------|
| `GeneralAssistant` | General-purpose conversational agent |
| `CodeHelper` | Programming and debugging |
| `Researcher` | Analytical research and summaries |
| `Writer` | Creative writing assistance |
| `DataAnalyst` | Data analysis and statistics |
| `DevOpsEngineer` | CI/CD and infrastructure |
| `CustomerSupport` | Customer inquiry handling |
| `Tutor` | Educational step-by-step explanations |
| `APIDesigner` | RESTful API design |
| `MeetingNotes` | Meeting transcript summarization |

### `security` — Security Feature Names

Short labels for the nine defense-in-depth features displayed on the dashboard:

```
merkleAudit, taintTracking, wasmSandbox, gcraRateLimit,
ed25519Signing, ssrfProtection, secretZeroize, loopGuard, sessionRepair
```

### `schedulerPage.cron` — Cron Presets

Human-readable labels for quick cron presets (23 entries), ranging from `everyMinute` through `mondaysMidnight`. These map to cron expressions in the scheduler UI.

### `overview.action*` — Activity Feed Actions

Standardized action type labels for the dashboard activity feed, such as `actionAgentCreated`, `actionToolUsed`, `actionNetworkAccess`, `actionLoginAttempt`, `actionDenied`, etc.

## How the Locale Files Connect to the Dashboard

```mermaid
graph LR
    A[locales/en.json] --> B[i18next init]
    C[locales/ja.json] --> B
    B --> D[React components]
    D -->|"t(\"nav.agents\")"| E[Rendered UI text]
    F[Browser language] -->|detection| B
    G[User preference] -->|override| B
```

1. **Initialization** — The i18next library (`i18n-4qyNqnlA.js`) loads locale files during dashboard startup.
2. **Language detection** — The browser's language preference is detected; fallback chain resolves to `en` if the user's language has no matching file.
3. **Key resolution** — When a component calls `t("agentsPage.agentSpawned", { name: "researcher" })`, i18next:
   - Looks up `agentsPage.agentSpawned` in the active locale
   - Falls back to `en.json` if missing
   - Interpolates `{name}` → `Agent "researcher" spawned`
4. **Parity validation** — The `dashboard/scripts/i18n-parity.mjs` script verifies that all keys in `en.json` exist in other locale files, preventing missing translations.

## Adding a New Language

1. Copy `locales/en.json` to `locales/<code>.json` (e.g., `fr.json`).
2. Translate all string values — **do not** modify keys.
3. Preserve all `{variable}` interpolation markers exactly.
4. Keep nested structure identical.
5. Run the i18n-parity script to verify no keys are missing or extra:
   ```bash
   node dashboard/scripts/i18n-parity.mjs
   ```

## Adding New UI Strings

1. Add the key to `locales/en.json` under the appropriate domain. Use nested structure for sub-groups:
   ```json
   "myFeature": {
     "title": "My Feature",
     "description": "Does something useful",
     "errorPrefix": "Error: {message}"
   }
   ```
2. Add the same key to all other locale files with translated values.
3. Reference in React components:
   ```js
   const { t } = useTranslation();
   <h2>{t("myFeature.title")}</h2>
   <p>{t("myFeature.errorPrefix", { message: err })}</p>
   ```

### Naming Conventions

| Pattern | Usage | Example |
|---------|-------|---------|
| `camelCase` | Standard keys | `agentSpawned` |
| `PascalCase` | Template/type names | `GeneralAssistant` |
| `snake_case` | Security feature IDs | `path_traversal` |
| `<domain>2` | Extended variant | `agentChat2` |

- **Toast messages**: Use past-tense or result-oriented (`modelSwitched`, `sessionDeleted`).
- **Error strings**: Include `{message}` interpolation for backend error details.
- **Page titles**: Match the `nav` key but live under the page domain.
- **Button labels**: Place reusable ones under `btn`; page-specific ones under the page domain.