# ⚙️ Configuration reference

> Every option you can set — in your `.sdtproj`, in VS Code, and through environment variables.

**On this page:** [Where options live](#where-options-live) · [Compare options](#compare-options) · [Deployment options](#deployment-options) · [Safety gates](#safety-gates) · [Pattern grammar](#pattern-grammar) · [VS Code settings](#vs-code-settings) · [Environment variables](#environment-variables)

---

## Where options live

`.sdtproj` deploy options live in two places, with the more specific one winning:

1. **Project-level** — under `deployOptions` in the `.sdtproj`. Applies to every profile unless overridden.
2. **Profile-level** — under `deploymentProfiles.<env>.deployOptions`. Merges on top of project-level.

Scalar values are overwritten; arrays are replaced wholesale (not merged) — a profile that sets `doNotDropObjectTypes: []` truly disables that protection. Defaults are production-safe; loosen them explicitly per environment.

```jsonc
{
  "name": "AnalyticsDb",
  "deployOptions": {
    "compare": { "ignoreComments": true, "ignoreCase": true },
    "deployment": { "blockOnPossibleDataLoss": true, "preDeployClone": true }
  },
  "deploymentProfiles": {
    "dev": {
      "connection": "snowflake-dev",
      "deployOptions": {
        "deployment": { "blockOnPossibleDataLoss": false, "allowDropTable": true }
      }
    }
  }
}
```

---

## Compare options

These live under `deployOptions.compare` and decide which field changes get surfaced. A `true` value means **ignore this dimension** — don't emit migration SQL for changes in it.

### Whitespace / cosmetic

| Option | Default | Effect |
|---|---|---|
| `ignoreComments` | `false` | Treat `comment` field differences as no-op. |
| `ignoreCase` | `false` | Compare unquoted identifiers case-insensitively. |
| `ignoreQuotedIdentifiers` | `false` | Treat `"FOO"` and `FOO` as equivalent when they resolve the same. |
| `ignoreWhitespace` | `false` | Ignore whitespace inside view / procedure / function bodies. |
| `ignoreSemicolons` | `false` | Ignore trailing-semicolon differences inside view / procedure bodies. |

### Column-level

| Option | Default | Effect |
|---|---|---|
| `ignoreColumnOrder` | `false` | Ignore column moves / ordinal changes. |
| `ignoreColumnDefaults` | `false` | Ignore `DEFAULT` expression differences. |
| `ignoreAutoincrementSeed` | `false` | Ignore AUTOINCREMENT seed differences. |
| `ignoreAutoincrementStep` | `false` | Ignore AUTOINCREMENT step differences. |
| `ignoreColumnPatterns` | `[]` | Wildcard list (`<DB>.<SCHEMA>.<TABLE>.<COLUMN>`) — these columns are skipped end-to-end. Use for housekeeping columns like `LOAD_DATE`. |

### Storage / retention

| Option | Default | Effect |
|---|---|---|
| `ignoreRetentionTime` | `false` | Ignore `DATA_RETENTION_TIME_IN_DAYS` differences. |
| `ignoreChangeTracking` | `false` | Ignore `CHANGE_TRACKING` flag differences (careful — streams). |
| `ignoreClusterBy` | `false` | Ignore clustering-key differences. |
| `ignoreSearchOptimization` | `false` | Ignore `SEARCH OPTIMIZATION` differences. |

### Governance / sidecars

| Option | Default | Effect |
|---|---|---|
| `ignoreTagAttachments` | `false` | Ignore tag attachment differences. |
| `ignorePolicyAttachments` | `false` | Ignore masking / row-access / etc. attachments. |
| `ignoreGrants` | **`true`** | Ignore direct grants (managed elsewhere by default). |
| `ignoreOwner` | **`true`** | Ignore `OWNER` (RBAC, not DDL, manages this). |
| `ignoreRoleMembership` | **`true`** | Ignore role-membership-grant changes. |
| `ignoreNetworkPolicyAttachment` | `false` | Ignore NETWORK POLICY attachments on users/accounts. |

### Filters

| Option | Default | Effect |
|---|---|---|
| `excludeObjectTypes` | `[]` | Skip the listed object types entirely. |
| `excludeObjectPatterns` | `[]` | Skip FQN-matching wildcard patterns. |
| `includeObjectPatterns` | `[]` | Whitelist — only objects matching at least one pattern are compared (empty = include everything). |
| `ignoreFieldPaths` | `[]` | Dotted-path field-level ignore (e.g. `columns.<>.dataType`). |

---

## Deployment options

These live under `deployOptions.deployment` and decide what SQL the emitter is allowed to produce.

### Top-level safety

| Option | Default | Effect |
|---|---|---|
| `blockOnPossibleDataLoss` | **`true`** | Refuse to deploy if any change is destructive / unrecoverable. |
| `blockWhenDriftDetected` | `false` | Refuse to deploy if the live target has changes not on the source side. |
| `treatWarningsAsErrors` | `false` | CI-strict mode — any WARNING blocks the deploy. |
| `verifyDeployment` | **`true`** | Run a verification pass before emitting SQL. |

### Drop policy

| Option | Default | Effect |
|---|---|---|
| `preserveTargetOnlyObjects` | `false` | Never emit DROPs for objects on target but not source. |
| `doNotDropObjectTypes` | `["STREAM","PIPE","TASK","EXTERNAL_TABLE"]` | Hard-skip drops for these types regardless of source. |
| `doNotDropObjectPatterns` | `[]` | Wildcard FQN list — these objects never get dropped. |
| `dropTargetOnlyPermissions` | `false` | Drop direct grants present on target but not source. |
| `dropTargetOnlyTagAttachments` | `false` | Drop tag attachments present on target but not source. |
| `doNotAlterChangeDataCaptureObjects` | **`true`** | Refuse drops/alters on objects feeding a stream/CDC pipeline. |
| `doNotAlterReplicatedObjects` | **`true`** | Refuse drops/alters on REPLICATION_GROUP / FAILOVER_GROUP objects. |

### Pre / post-deploy hooks

| Option | Default | Effect |
|---|---|---|
| `preDeployScriptPatterns` | `["scripts/pre/*.sql"]` | `.sql` files executed before the generated migration body. |
| `postDeployScriptPatterns` | `["scripts/post/*.sql"]` | `.sql` files executed after the generated migration body. |

Use pre-deploy scripts for snapshotting, warming caches, granting transient privileges, or suspending downstream tasks; use post-deploy for seeding reference data, resuming tasks, or refreshing dynamic tables.

### Snowflake-specific

| Option | Default | Effect |
|---|---|---|
| `pauseTasksBeforeDeploy` | `true` | Suspend downstream tasks before deploying STREAM/TASK changes. |
| `refreshDynamicTablesAfterDeploy` | `false` | Post-flight refresh for changed dynamic tables. |
| `preDeployClone` | `false` | Zero-copy clone the target before deploying (rollback safety). |
| `preDeployCloneName` | `"{db}_PREDEPLOY_{ts}"` | Clone name template (`{db}` = db name, `{ts}` = `YYYYMMDD_HHMMSS`). |
| `resumeWarehousesBeforeDeploy` | `true` | Resume SUSPENDED warehouses the deploy needs. |
| `includeTransactionalScripts` | `false` | Wrap with `BEGIN … COMMIT` (rarely needed — Snowflake DDL is auto-commit). |
| `generateSmartDefaults` | `true` | Emit sensible defaults for new NOT NULL columns so existing rows don't error. |
| `scriptBanner` | `""` | Banner text appended at the top of the generated migration script. |
| `strategyOverrides` | `{}` | Per-type override, e.g. `{ "STREAM": { "rebuild": "WARN" } }`. |

---

## Safety gates

These per-operation gates live under `deployOptions.deployment` and unlock blocked operations. All default to `false` (except `allowExternalLanguageChanges`).

| Option | Default | Unlocks |
|---|---|---|
| `allowDropTable` | `false` | `DROP TABLE`. |
| `allowDropColumn` | `false` | Column drops. |
| `allowNarrowingTypes` | `false` | Narrowing column type changes (`VARCHAR(255)` → `VARCHAR(50)`). |
| `allowTableRebuild` | `false` | Rebuilds where the strategy is `BLOCKED` (clustering changes, etc.). |
| `allowUnrecoverableDrop` | `false` | Drops of stream / pipe / task / similar unrecoverable types. |
| `allowExternalLanguageChanges` | `true` | Java / Python UDF / procedure changes. |
| `noRebuildObjectPatterns` | `[]` | Object FQN patterns exempt from rebuild even if the strategy allows. |

See [The safety classifier](safety-classifier.md) for how these interact with the four risk categories.

---

## Pattern grammar

`excludeObjectPatterns`, `includeObjectPatterns`, `ignoreColumnPatterns`, `doNotDropObjectPatterns`, and `noRebuildObjectPatterns` all use the same wildcard grammar:

- `*` matches any run of identifier characters (alphanumeric + underscore).
- `?` matches exactly one identifier character.
- `.` is literal.
- Patterns are case-insensitive.

| Pattern | Matches |
|---|---|
| `ANALYTICS.*.*` | Every schema and object inside `ANALYTICS`. |
| `ANALYTICS.AUDIT.*` | Every object inside `ANALYTICS.AUDIT`. |
| `*.*.RAW_*` | Every table whose name starts with `RAW_`. |
| `ANALYTICS.GOLD.FACT_SALES.LOAD_DATE` | Just that single column. |
| `*.*.*.LOAD_DATE` | Every `LOAD_DATE` column on every table. |

---

## VS Code settings

Configure the SDT extension under **Settings → Extensions → SDT** (or in `settings.json`). All keys are prefixed `sdt.`.

| Setting | Default | What it does |
|---|---|---|
| `sdt.connectionProfilesPath` | `""` | Path to the connection profiles file. Defaults to `~/.sdt/profiles.json`. |
| `sdt.defaultProfile` | `""` | Default Snowflake connection profile name. |
| `sdt.compare.ignoreCase` | `false` | Treat unquoted identifiers case-insensitively when comparing. |
| `sdt.deploy.useClonesForRollback` | `true` | Pre-create a zero-copy clone before deployment so rollback can be done via `SWAP WITH`. |
| `sdt.formatOnSave` | `true` | Auto-format `.sql` files on save with the SDT formatter. Disable to use VS Code's built-in format-on-save. |
| `sdt.discoverability.level` | `standard` | Coarse discoverability knob. **minimal** suppresses hints, rich tooltips, CodeLens, and context-menu surfaces; **standard** is the default; **verbose** adds inline Try-it buttons. |
| `sdt.discoverability.hints` | `auto` | Per-channel override for post-command hints. `auto` follows the level; `on`/`off` force it. |
| `sdt.discoverability.tooltips` | `auto` | Per-channel override for rich Markdown tooltips on Object Explorer nodes. |
| `sdt.discoverability.codeLens` | `auto` | Per-channel override for SDT CodeLens hints. |
| `sdt.discoverability.contextMenu` | `auto` | Per-channel override for feature-aware right-click menu entries. |
| `sdt.discoverability.emptyStates` | `auto` | Per-channel override for actionable empty-state placeholders. |
| `sdt.discoverability.suppressedHintIds` | `[]` | Hint IDs silenced via "Don't show this again". Managed automatically. |
| `sdt.licenseServer.url` | `""` | Base URL of your organization's On-prem License Server (Enterprise SSO). |
| `sdt.licenseServer.oidcClientId` | `""` | OAuth2 client ID for desktop sign-in (Enterprise SSO). |
| `sdt.licenseServer.oidcAuthorizeUrl` | `""` | Identity provider authorization endpoint (Enterprise SSO). |
| `sdt.licenseServer.oidcTokenUrl` | `""` | Identity provider token endpoint (Enterprise SSO). |
| `sdt.licenseServer.scopes` | `[]` | OAuth scopes for SSO sign-in. Empty uses `openid profile email`. |

---

## Environment variables

| Variable | Effect |
|---|---|
| `NO_COLOR` | Disables ANSI colour in CLI output (per the [NO_COLOR convention](https://no-color.org/)). |
| `CI` | When `CI=true`, the CLI runs non-interactively — it never prompts (require `--yes`), never opens a browser, and disables spinners. |
| `SDT_NO_HINTS` | `SDT_NO_HINTS=1` maps to the `minimal` discoverability level. |

Connection profiles also support `env:VAR_NAME` placeholders so secrets stay out of `profiles.json`.

---

**Next:** [CLI reference](cli-reference.md) · **Up:** [Documentation home](README.md)
