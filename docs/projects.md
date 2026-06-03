# 📁 Projects & suites

> Author a Snowflake schema as a folder of `.sql` files, build it into a portable artifact, and stitch multiple projects into one deploy bundle.

**On this page:** [The project file](#the-project-file) · [Folder layout](#folder-layout) · [Project scope](#project-scope) · [Variables & deployment profiles](#variables--deployment-profiles) · [Pre/post deploy scripts](#prepost-deploy-scripts) · [Building a `.sdtpac`](#building-a-sdtpac) · [Slices](#slices--owning-part-of-a-shared-account) · [Suites](#suites--grouping-projects)

---

## The project file

An SDT (Snowflake Data Tools) project is a folder of `.sql` files — one object per file — described by a single JSON manifest with the `.sdtproj` extension. The manifest declares the project's identity, what part of the account it owns, and how it deploys to each environment.

A minimal `MyProject.sdtproj`:

```json
{
  "$schema": "https://sdt.dev/schemas/sdtproj/v1.json",
  "name": "AnalyticsDb",
  "version": "1.0.0",
  "targetPlatform": { "platform": "Snowflake" },
  "scope": { "type": "database", "database": "ANALYTICS" }
}
```

A fuller example using most fields:

```json
{
  "$schema": "https://sdt.dev/schemas/sdtproj/v1.json",
  "name": "AnalyticsDb",
  "version": "1.0.0",
  "targetPlatform": {
    "platform": "Snowflake",
    "edition": "Enterprise",
    "minBundle": "2026_01"
  },
  "scope": {
    "type": "database",
    "database": "ANALYTICS"
  },
  "include": ["**/*.sql", "**/*.json"],
  "exclude": ["bin/**", "obj/**"],
  "defaults": {
    "dataRetentionTimeInDays": 1,
    "defaultDdlCollation": "en-ci",
    "warehouse": "COMPUTE_WH"
  },
  "preDeployScripts": ["scripts/pre/*.sql"],
  "postDeployScripts": ["scripts/post/*.sql"],
  "deploymentProfiles": {
    "dev":  { "connection": "dev",  "variables": { "ENV": "DEV" },  "cloneForRollback": true },
    "prod": { "connection": "prod", "variables": { "ENV": "PROD" }, "cloneForRollback": true }
  }
}
```

> [!TIP]
> The `$schema` field points your editor at the published JSON schema, so VS Code gives you field autocomplete and validation as you type.

### Fields

| Field | What it does | Notes / default |
|---|---|---|
| `$schema` | URL of the JSON schema | Triggers IDE autocomplete |
| `name` | Logical project name | Required. Used for the built `.sdtpac` filename |
| `version` | Semver-style version | Required. Bumped manually |
| `targetPlatform` | Compatibility hints (`platform`, `edition?`, `minBundle?`) | Required. `edition` and `minBundle` are advisory |
| `scope` | What the project owns — see [Project scope](#project-scope) | Required |
| `include` | Glob patterns for source files | Default `["**/*.sql","**/*.json"]` |
| `exclude` | Globs that override `include` | Default `["bin/**","node_modules/**",".sdt-cache/**"]` |
| `defaults` | Defaults applied to objects that don't set them (`dataRetentionTimeInDays`, `defaultDdlCollation`, `warehouse`) | Optional |
| `preDeployScripts` | Globs run before the migration script | Optional. Order: alpha by path |
| `postDeployScripts` | Globs run after the migration script | Optional |
| `deploymentProfiles` | Named deploy targets — see [Variables & deployment profiles](#variables--deployment-profiles) | Optional |

The schema is strict — unknown fields are rejected at load time, so typos surface immediately.

---

## Folder layout

The folder layout is a convention: SDT maps each file's path to an object's identity (database / schema / type / name) rather than reading the file first. Keep files in their canonical folders.

```
MyProject/
├── MyProject.sdtproj
├── account/                          (only if scope = "account")
│   ├── warehouses/
│   ├── roles/
│   ├── users/
│   ├── network_policies/
│   └── integrations/
│       ├── api/
│       └── storage/
├── databases/
│   └── ANALYTICS/
│       ├── database.sql              (CREATE OR ALTER DATABASE ANALYTICS ...)
│       └── schemas/
│           └── PUBLIC/
│               ├── schema.sql
│               ├── tables/
│               │   ├── DIM_CUSTOMER.sql
│               │   └── FACT_SALES.sql
│               ├── views/
│               ├── procedures/
│               ├── functions/
│               ├── streams/
│               ├── tasks/
│               ├── dynamic_tables/
│               ├── stages/
│               ├── file_formats/
│               ├── sequences/
│               ├── tags/
│               └── policies/
│                   ├── masking/
│                   └── row_access/
├── scripts/
│   ├── pre/
│   └── post/
└── bin/                              (build output — gitignored)
```

Each `.sql` file contains **exactly one** object definition, ideally as a `CREATE OR ALTER ...` statement (or `CREATE OR REPLACE` where Snowflake doesn't support `CREATE OR ALTER` yet).

> [!IMPORTANT]
> For state-bearing types (STREAM, PIPE, TASK), edit carefully — the safety classifier flags drop-and-recreate operations as UNRECOVERABLE. See the [Safety classifier](safety-classifier.md).

---

## Project scope

`scope` is a hard fence: `extract` only walks objects inside it, and `compare` ignores anything outside it. A tighter scope means a faster compare and a safer deploy.

| Scope `type` | What the project owns | Required fields |
|---|---|---|
| `account` | Account-level objects (warehouses, roles, users, integrations, network policies) plus databases | `type` only |
| `database` | One database and everything inside it | `database` |
| `schema` | One schema | `database` + `schema` |

```json
{ "scope": { "type": "schema", "database": "ANALYTICS", "schema": "PUBLIC" } }
```

---

## Variables & deployment profiles

A **deployment profile** pairs a connection name with a set of variables for one environment. Profiles live under `deploymentProfiles` in the `.sdtproj`; the `connection` name resolves to a profile in `~/.sdt/profiles.json`.

```json
{
  "deploymentProfiles": {
    "dev": {
      "connection": "dev-profile",
      "variables": { "ENV_PREFIX": "DEV_", "RETENTION_DAYS": "1", "WAREHOUSE": "DEV_WH" }
    },
    "prod": {
      "connection": "prod-profile",
      "variables": { "ENV_PREFIX": "", "RETENTION_DAYS": "30", "WAREHOUSE": "PROD_WH" }
    }
  }
}
```

### `$(VAR)` substitution

In raw `.sql` files, write `$(VAR_NAME)` and the value is substituted into the generated migration script at deploy time. Substitution is textual — wrap quoted strings yourself when needed.

```sql
ALTER WAREHOUSE $(WAREHOUSE) SET WAREHOUSE_SIZE = 'X-SMALL';

CREATE OR ALTER TABLE $(ENV_PREFIX)ORDERS (
  id INT,
  ts TIMESTAMP_NTZ
)
  DATA_RETENTION_TIME_IN_DAYS = $(RETENTION_DAYS);
```

`$(VAR)` is wired into `sdt script --variables KEY=VALUE,...` and `sdt promote` (which passes the profile's variables automatically).

### Resolution order

Later sources win:

1. Built-in variables (below).
2. Project-level `defaults`.
3. Deployment-profile `variables`.
4. `--variables KEY=VALUE,...` on `sdt script`.

### Built-in variables

| Name | Value |
|---|---|
| `SDT_PROFILE` | The active profile name |
| `SDT_PLATFORM` | Always `snowflake` for SDT |
| `SDT_USER` | The Snowflake user that ran the deploy |
| `SDT_TIMESTAMP` | ISO 8601 UTC at deploy start |
| `SDT_PROJECT_NAME` | From `.sdtproj.name` |
| `SDT_PROJECT_VERSION` | From `.sdtproj.version` |

---

## Pre/post deploy scripts

`preDeployScripts` run before the generated migration body; `postDeployScripts` run after it. Both take glob lists and run in alphabetical order by path. Use them for grants, warehouse resizing, or data backfills that bracket the schema change.

```json
{
  "preDeployScripts": ["scripts/pre/*.sql"],
  "postDeployScripts": ["scripts/post/*.sql"]
}
```

`$(VAR)` substitution applies inside these scripts just as it does in object files.

---

## Building a `.sdtpac`

A `.sdtpac` is a portable, self-contained build artifact of your project — a ZIP container you can hand to CI, sign, archive, or deploy without the original source tree.

```sh
sdt build ./MyProject.sdtproj
# Built ./bin/AnalyticsDb.sdtpac   (named from the project's "name" field)
```

A `.sdtpac` holds four sections:

| Entry | What it contains |
|---|---|
| `manifest.json` | Project metadata — name, version, scope, target platform, build timestamp, object count, format version |
| `model.json` | The canonical normalized array of Snowflake objects, each tagged with `objectType` and a fully-qualified `fqn` |
| `source/` | Your original `.sql` files, preserved byte-for-byte |
| `checksums.json` | A SHA-256 per object (`<objectType>:<fqn>`) for fast drift detection |

Why it exists: the pac is the unit of deployment in CI/CD. `sdt compare` and `sdt publish` both accept a `.sdtpac` as a source, so you build once, then deploy that exact artifact to every environment. Because the source is preserved alongside the normalized model, a deploy can re-emit your hand-written DDL where the model round-trip would lose detail.

> [!NOTE]
> The pac format is versioned. A reader rejects a pac whose format version it doesn't understand rather than guessing — rebuild it with a matching version of the CLI.

---

## Slices — owning part of a shared account

When several teams share one Snowflake account, a full-scope project is dangerous: `publish` would propose dropping anything on the target that isn't in your source. A **Slice** narrows a project to only the objects it owns.

Add a `slice` block to the `.sdtproj`:

```json
{
  "slice": {
    "owns": ["MARKETING.*", "ANALYTICS.MARKETING_*"],
    "reads": ["FINANCE.dim_account", "GOVERNANCE.tag.*"]
  }
}
```

| Field | What it does |
|---|---|
| `owns` | Glob patterns matched against the FQN. Matching objects are in scope for diff and deploy |
| `reads` | Read-only references the project may mention (foreign keys, view dependencies) but never mutate |
| `caseSensitive` | When `false`, matches case-insensitively. Default is `true` for SDT |

Glob rules: `*` matches within one identifier segment, `**` matches across segments, plain identifiers match literally.

With a Slice in place:

- Objects on the target that don't match `owns` are partitioned out of the diff — they never show as added, removed, or modified.
- Objects matching `reads` but not `owns` are read-only — the engine refuses to emit any DDL touching them.
- `sdt compare` prints an informational footer listing what was left untouched (for example, `FINANCE.* exists on the target but is outside this slice`).

| Command | Slice behavior |
|---|---|
| `sdt compare` | Honors `slice` by default. Pass `--no-slice` to treat the project as full-scope |
| `sdt publish` | Honors `slice` by default. `--no-slice` works the same way |
| `sdt validate` | Compiles the slice and flags any overlap between `owns` and `reads` patterns |

---

## Suites — grouping projects

A **Suite** is a `.sdtsuite` file that aggregates multiple Slice-aware projects into one deploy bundle and validates that they fit together cleanly.

```json
{
  "$schema": "https://sdt.dev/schemas/sdtsuite/v1.json",
  "name": "AcmeDataPlatform",
  "version": "1.0.0",
  "projects": [
    { "path": "./marketing/MarketingAnalytics.sdtproj" },
    { "path": "./finance/FinanceCore.sdtproj" },
    { "path": "./governance/Governance.sdtproj" }
  ]
}
```

Scaffold and validate a suite:

```sh
sdt suite init --output ./Platform.sdtsuite       # scaffold a new suite
sdt suite validate --suite ./Platform.sdtsuite    # run cross-project checks
```

`sdt suite validate` runs five cross-project validations:

1. **Overlap detection** — no two slices' `owns` patterns may intersect. Two projects claiming the same FQN is an error.
2. **Reads-without-owner** — every `reads` pattern should match some other project's `owns`. Orphan reads are warnings.
3. **Cycle detection** — project A reading from B which reads from A is an error.
4. **Connection consistency** — every project in a suite must target the same connection profile per environment. Mixed environments are rejected.
5. **Schema overlap** — compiled membership functions are tested against a sample fixture, so concrete overlaps surface even when globs don't visibly intersect.

> [!TIP]
> Both Slices and Suites are Pro-tier features. During the public beta, all paid features are unlocked.

### In VS Code

The extension ships a Suite tree view in the activity bar showing every open `.sdtsuite` and the projects inside. Run **SDT: Validate Suite** from the Command Palette to run the five checks and see findings inline in the Problems panel. Project entries carry a `(scoped)` badge when they have a `slice`, or `(full)` when they own the entire target.

---

**Next:** [Extract](extract.md) · **Up:** [Documentation home](README.md)
