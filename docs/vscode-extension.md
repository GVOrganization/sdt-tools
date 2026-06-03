# 🧩 VS Code extension reference

> Everything the Snowflake Data Tools extension adds to VS Code — activity-bar views, every Command Palette command, right-click menus, settings, and the project editor.

**On this page:** [Activity bar views](#activity-bar-views) · [Command Palette](#command-palette) · [Context menus](#context-menus) · [Settings](#settings) · [Keybindings](#keybindings) · [Walkthrough](#walkthrough) · [Project editor](#project-editor)

---

The SDT extension activates when you open a `.sql` file, a `.sdtproj`, or a `.sdtsuite` workspace. It adds a dedicated activity-bar container, **SDT — Snowflake Data Tools**, and wires the full author → compare → deploy loop into the editor. The companion `sdt` CLI does the same work headlessly — see the [CLI reference](cli-reference.md).

---

## Activity bar views

Click the SDT icon in the activity bar to open five tree views.

| View | What it shows |
|---|---|
| **Projects** | Every `.sdtproj` in the open workspace. Right-click a project to build, publish, validate, open compare profiles, or drift-check. |
| **Suites** | Every `.sdtsuite` (multi-project Suite) in the workspace, with a refresh action. |
| **Snowflake Connections** | Configured connection profiles. Right-click a connection to test it or extract it into a project. |
| **Object Explorer** | The live database → schema → object tree for a connected target, read from the local catalog cache. Right-click objects for refactors, lineage, extract, and compare. |
| **Deploy History** | Past deploys with status-badge icons; refreshable. |

> [!TIP]
> Object Explorer reads a cached snapshot, so it never issues live queries while you browse. Refresh the cache from the terminal with `sdt catalog refresh --connection <name>`.

---

## Command Palette

Open the Command Palette (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>) and type **SDT:** to see every command.

### Project

| Command | What it does |
|---|---|
| **SDT: New Project** | Scaffold a new `.sdtproj` with the canonical directory layout. |
| **SDT: Open Project** | Open an existing project. |
| **SDT: Build Project (.sdtpac)** | Compile the project to a `.sdtpac` artifact. |
| **SDT: Publish Project** | Deploy the project to a connection. |
| **SDT: Validate Project** | Run the schema-shape + reference checks. |
| **SDT: Validate Suite** | Validate a multi-project `.sdtsuite`. |

### Compare & deploy

| Command | What it does |
|---|---|
| **SDT: Extract Snowflake Account → Project** | Reverse-engineer a live account into project `.sql` files. |
| **SDT: Schema Compare** | Diff project / pac / live account and emit a safe migration script. |
| **SDT: Drift Check** | Compare the live warehouse to the project; report only. |
| **SDT: Open Compare Profiles** | Manage saved source → target compare profiles. |
| **SDT: Open Interactive Schema Diagram** | Open the interactive schema diagram. |

### Connections

| Command | What it does |
|---|---|
| **SDT: Add Snowflake Connection** | Add a profile to `~/.sdt/profiles.json`. |
| **SDT: Test Snowflake Connection** | Verify a profile connects. |
| **SDT: Refresh Connections** | Reload the Connections view. |

### Object Explorer

| Command | What it does |
|---|---|
| **SDT: Refresh Object Explorer** | Reload the live object tree from the cache. |
| **SDT: Copy FQN** | Copy an object's fully-qualified name. |
| **Rename Column…** | Record a column rename refactor on a table. |
| **Change Column Type…** | Record a column type-change refactor. |
| **Drop Column…** | Record a column drop refactor. |

### AI

| Command | What it does |
|---|---|
| **SDT: Sketch Object from Description (AI)** | Scaffold idiomatic DDL from a prose description. |
| **SDT: Suggest Safer Alternative (AI)** | Propose a safer DDL alternative for a dangerous change. |
| **SDT: Show Column Lineage** | Trace column-level lineage for a view or table. |
| **SDT: Open Ask SDT chat panel (AI)** | Open the "Ask SDT" chat panel. |
| **SDT: Apply chat-proposed refactor ops** | Apply refactor operations proposed in the chat panel. |
| **SDT: Open Visual Table Designer** | Open the visual table designer. |

### Formatting

| Command | What it does |
|---|---|
| **SDT: Format SQL Document** | Format the active `.sql` file with the SDT formatter. |

### Discovery & support

| Command | What it does |
|---|---|
| **SDT: Find a Feature** | Search every SDT feature (<kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd>). |
| **SDT: Open Query Results panel** | Open the query-results pane. |
| **SDT: Sign in with SSO (Enterprise)** | Sign in via your org's license server. |
| **SDT: Sign out (SSO)** | Sign out of SSO. |
| **SDT: Show License Status** | Show the current license status. |
| **SDT: Report a Bug…** | Open the bug-report flow. |
| **SDT: Refresh Deploy History** | Reload the Deploy History view. |

---

## Context menus

SDT adds right-click actions that are context-aware — they appear only where they apply.

| Where you right-click | Available actions |
|---|---|
| **A table in Object Explorer** (TABLE, EXTERNAL_TABLE, DYNAMIC_TABLE, HYBRID_TABLE, ICEBERG_TABLE) | Rename Column…, Change Column Type…, Drop Column…, Suggest Safer Alternative (AI) |
| **A view in Object Explorer** (VIEW, MATERIALIZED_VIEW, DYNAMIC_TABLE) | Show Column Lineage, Suggest Safer Alternative (AI) |
| **A database or schema in Object Explorer** | Extract → Project, Schema Compare |
| **A connection in Object Explorer** | Test Connection, Extract → Project, Refresh Deploy History |
| **A project in the Projects view** | Build, Publish, Validate, Open Compare Profiles, Drift Check |
| **A connection in the Connections view** | Test Connection, Extract → Project |
| **In a `.sql` editor** | Format SQL Document, Suggest Safer Alternative (AI), Show Column Lineage, Open Ask SDT chat panel (AI) |

---

## Settings

Configure SDT under **Settings → Extensions → SDT — Snowflake Data Tools**, or edit `settings.json` directly.

| Setting | Default | What it does |
|---|---|---|
| `sdt.connectionProfilesPath` | `""` | Path to the connection profiles file. Empty uses `~/.sdt/profiles.json`. |
| `sdt.defaultProfile` | `""` | Default connection profile name. |
| `sdt.compare.ignoreCase` | `false` | Treat unquoted identifiers case-insensitively when comparing. |
| `sdt.deploy.useClonesForRollback` | `true` | Pre-create a zero-copy clone before deploy so rollback can use SWAP WITH. |
| `sdt.formatOnSave` | `true` | Auto-format `.sql` files on save with the SDT formatter. |
| `sdt.discoverability.level` | `standard` | Coarse discoverability knob: `minimal` suppresses hints/tooltips/CodeLens/context surfaces; `standard` is default; `verbose` adds inline Try-it buttons. |
| `sdt.discoverability.hints` | `auto` | Per-channel override for post-command hints. `auto` follows the level. |
| `sdt.discoverability.tooltips` | `auto` | Per-channel override for rich tooltips on Object Explorer nodes. |
| `sdt.discoverability.codeLens` | `auto` | Per-channel override for SDT CodeLens hints. |
| `sdt.discoverability.contextMenu` | `auto` | Per-channel override for feature-aware right-click entries. |
| `sdt.discoverability.emptyStates` | `auto` | Per-channel override for actionable empty-state placeholders (stays on even at `minimal`). |
| `sdt.discoverability.suppressedHintIds` | `[]` | Hint IDs silenced via "Don't show this again"; managed automatically. |
| `sdt.licenseServer.url` | `""` | Base URL of your org's SDT On-prem License Server (Enterprise SSO). |
| `sdt.licenseServer.oidcClientId` | `""` | OAuth2 client ID for desktop sign-in (Enterprise SSO). |
| `sdt.licenseServer.oidcAuthorizeUrl` | `""` | Identity-provider authorization endpoint (Enterprise SSO). |
| `sdt.licenseServer.oidcTokenUrl` | `""` | Identity-provider token endpoint (Enterprise SSO). |
| `sdt.licenseServer.scopes` | `[]` | OAuth scopes for SSO. Empty uses `openid profile email`. |

---

## Keybindings

| Keybinding | Command |
|---|---|
| <kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd> (mac: <kbd>Cmd</kbd>+<kbd>K</kbd> <kbd>F</kbd>) | **SDT: Find a Feature** |

> [!TIP]
> "Find a Feature" is the in-editor equivalent of `sdt find` — when you can't remember a command name, press <kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd> and search.

---

## Walkthrough

The first time you open VS Code with SDT installed, **Get started with SDT** appears on the Welcome page (also reachable via **Help → Welcome**). It walks you through:

1. **Add a Snowflake connection** — register a profile.
2. **Scaffold a new project** — create a `.sdtproj` with the canonical layout.
3. **Extract a live account** — pull a live account's schema into `.sql` files.
4. **Compare and deploy** — diff your project against a target and emit a safe migration script.

---

## Project editor

`.sdtproj` files open in a dedicated **SDT Project** editor rather than as raw JSON, giving you a structured view of project scope, object folders, and deployment settings. Snowflake SQL files get syntax highlighting, the built-in snippet library, live lint squiggles, and column-validation diagnostics inside any `.sdtproj`.

---

**Next:** [AI features](ai-features.md) · **Up:** [Documentation home](README.md)
