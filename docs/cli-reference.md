# ⌨️ CLI reference

> The complete `sdt` command catalog, organized by the workflow you're in — author a project, connect, deploy safely, analyze, and ship through CI.

**On this page:** [Discovery](#discovery) · [Global flags](#global-flags) · [Exit codes](#exit-codes) · [Output formats](#output-formats) · [Project lifecycle](#project-lifecycle) · [Connections](#connections) · [Safety & rollback](#safety--rollback) · [Code quality](#code-quality) · [Analysis](#analysis) · [Data](#data) · [CI/CD](#cicd) · [AI](#ai) · [Migration & interop](#migration--interop) · [Utilities](#utilities)

---

## Discovery

You never have to memorize this page. Every command, flag, and config key is searchable from the terminal.

```sh
sdt explain "drop table"        # search the options catalog by topic
sdt explain --list              # dump every option
sdt find "drift detection"      # "what can SDT do for X?"
sdt <command> --help            # per-command usage + flags
```

> [!TIP]
> `sdt explain <anything>` and `sdt <command> --help` are the fastest path to discovery — start there before scanning the tables below.

> [!NOTE]
> A few commands are still being wired in the current beta. `sdt exec` and `sdt search` are headless-path stubs that today direct you to the VS Code query window or `sdt extract`. `sdt format` is functional but young. These are marked **(beta)** in the tables below.

---

## Global flags

These apply to most commands.

| Flag | What it does |
|---|---|
| `--output table\|json\|yaml\|sql` | Output format; defaults to `table` for humans. |
| `--explain` | Narrate the result through your configured AI provider (on supported commands). |
| `--quiet` / `--verbose` / `--debug` | Log verbosity. |
| `--no-color` | Disable ANSI color (also honors the `NO_COLOR` env var). |
| `--yes` / `--confirm` | Required for write operations in non-interactive mode (CI or piped stdin). |

Structured output goes to stdout; logs go to stderr. In non-interactive mode (no TTY or `CI=true`) SDT never prompts and never opens a browser — pass `--yes` explicitly.

---

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | User error (bad args, missing file, syntax error) |
| 2 | System error (network, permission, internal failure) |
| 3 | Compare diff found (CI drift detection, flag-controlled) |
| 4 | Destructive changes detected and not approved |
| 5 | Drift detected (`sdt drift` in CI mode) |
| 6 | Locked feature attempted without a Pro/Team/Enterprise license |

---

## Output formats

`--output table | json | yaml | sql` controls the global format. Several commands also take `--format text | json | markdown | sarif` (lint, diagnose) or `--format table | json | markdown` (compare, cost-estimate). SARIF output is consumable by GitHub code-scanning and Azure DevOps.

---

## Project lifecycle

The everyday loop: author a `.sdtproj`, build it to a `.sdtpac`, deploy it, observe what changed. See [Projects & suites](projects.md), [Extract](extract.md), [Schema compare](schema-compare.md), [Safe deploy](safe-deploy.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt init` | Scaffold a new project (`<name>.sdtproj` + folder layout). | `--name`, `--scope account\|database\|schema`, `--db`, `--schema`, `--dir` |
| `sdt validate` | Schema-shape check; with `--references` runs the build-time semantic resolver. | `--project`, `--references`, `--columns`, `--check-variables`, `--min-severity`, `--format`, `--out` |
| `sdt extract` | Reverse-engineer a live account into per-object `.sql` files. | `--connection`, `--types`, `--output` |
| `sdt build` | Compile a `.sdtproj` to a `.sdtpac` build artifact. | `-o, --out` |
| `sdt publish` | Deploy a `.sdtpac` to a connection; honors every safety gate. | `--pac`, `--connection`, `--dry-run`, `--apply`, `--yes`, `--manifest`, `--confirm-production`, `--profile`, `--variables`, `--restore-from-snapshot`, `--map`, `--map-file` |
| `sdt compare` | Diff any two of project / pac / live account. | `--format`, `--explain`, `--color`, `--ignore-case`, `--no-slice`, `--type-safe`, `--break-on`, `--write-impact`, `--map`, `--map-file`, `--no-history` |
| `sdt drift` | Compare the live warehouse to what the project expects; report only. | `--project`, `--connection` |
| `sdt script` | Generate the migration SQL without executing it (always offline). | `--source`, `--target`, `-o, --out`, `--profile`, `--variables`, `--map`, `--map-file` |
| `sdt promote` | Branch-per-env deploy automation; diff two live envs, emit a bundle, optionally open a PR. | `--from`, `--to`, `--open-pr`, `--repo`, `--base` |
| `sdt refresh` | Generate a post-deploy REFRESH script (dynamic tables + tasks). | `--source`, `-o`, `--no-dynamic-tables`, `--no-tasks` |
| `sdt exec` **(beta)** | Run a `.sql` script against one or more profiles in parallel. | `--profiles`, `--yes`, `--timeout`, `--format` |

---

## Connections

Manage Snowflake connection profiles in `~/.sdt/profiles.json`. See [Connections](connections.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt connection add` | Register a profile. | `--name`, `--account`, `--user`, `--auth PASSWORD\|KEY_PAIR\|OAUTH\|EXTERNAL_BROWSER\|MFA`, `--password`, `--key`, `--key-passphrase`, `--role`, `--warehouse`, `--database`, `--schema` |
| `sdt connection list` | List configured profiles. | — |
| `sdt connection test <name>` | Verify a profile connects. | — |
| `sdt connection remove <name>` | Delete a profile. | — |

> [!IMPORTANT]
> Secrets are never inlined. `--password` and `--key-passphrase` accept `env:VAR_NAME` placeholders that resolve at runtime.

---

## Safety & rollback

No silent destruction: destructive changes are gated, snapshots are captured, and reverts replay reverse SQL. See [Safety classifier](safety-classifier.md) and [Safe deploy](safe-deploy.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt safety` | Inspect the safety-finding catalog. | `list --category`, `explain <code>`, `--format` |
| `sdt revert` | Roll back a previous `sdt publish` using its captured manifest. | `--manifest`, `--connection`, `--yes`, `--dry-run`, `--continue-on-error` |
| `sdt snapshot` | Inspect and prune the pre-deploy snapshot registry. | `list`, `show <id>`, `prune`, `--apply`, `--yes`, `--out`, `--root`, `--registry-only`, `--expired-only` |
| `sdt replay` | Re-execute the forward SQL from deploy manifests against a target. | `--manifest`, `--connection`, `--yes`, `--filter`, `--from-step`, `--to-step`, `--continue-on-error`, `--dry-run` |
| `sdt approval` | File-based multi-approver gate for production deploys. *(Team)* | `add`, `list`, `clear`, `check`, `--required`, `--allowed`, `--digest`, `--root` |
| `sdt approval-chain` | Cryptographically-signed M-of-N approval workflow (Ed25519). *(Team)* | `keygen`, `sign`, `verify`, `--deploy-id`, `--env`, `--key`, `--config`, `--records` |
| `sdt verify` | Recompute checksums on a `.sdtpac` and confirm every object matches the manifest. | `--pac`, `--json` |
| `sdt purge` | Generate a DROP script for every object in a project (review-only). | `--source`, `--confirm-production`, `-o`, `--allow-drop-table`, `--allow-unrecoverable-drop`, `--allow-narrowing-types`, `--allow-table-rebuild` |
| `sdt data-fit` | Generate read-only probes confirming no rows overflow a narrowing type change. *(Pro)* | `--source`, `--target`, `-o`, `--format` |
| `sdt scan-secrets` | Scan DDL bodies for hard-coded credentials. | `--source`, `--strict`, `--format`, `-o` |
| `sdt rollback-suggest` | AI-propose a reverse-SQL skeleton for a forward DDL the deterministic revert couldn't invert (review-only). *(Team)* | `--forward-sql`, `--fqn`, `--object-type`, `--finding`, `--intent`, `--format` |

---

## Code quality

Lint, format, and enforce team SQL standards. See [Configuration reference](configuration.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt lint` | Lint a SQL script for risky deploy patterns. | `--script`, `--rules`, `--max-warnings`, `--format text\|json\|markdown\|sarif`, `--rules-dir`, `--dbt`, `--no-dbt`, `--no-custom-rules` |
| `sdt format` **(beta)** | Token-faithful, idempotent SQL formatter. | `--check`, `--in-place`, `-p, --project`, `--style compact\|expanded\|wide`, `--dbt`, `--no-dbt`, `--engine` |
| `sdt standards` | Team SQL standards: scaffold, lint, explain. | `init --preset`, `check`, `--fix`, `--ai-fix`, `--explain <rule>`, `--json` |
| `sdt install-hooks` | Install a pre-commit hook running `sdt validate` + `sdt lint` on changed `.sql` files. | — |

---

## Analysis

Understand what depends on what, where data flows, and what a change costs. See [Schema compare](schema-compare.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt lineage` | Data-flow lineage from view / procedure bodies and task references. | `--project`, `--fqn`, `--direction`, `--depth`, `--format mermaid\|json\|dot` |
| `sdt graph` | Emit the dependency graph as Mermaid / DOT. | `--project`, `--format`, `-o` |
| `sdt impact` | Estimate blast radius of a pending change. | `--project`, `--object` |
| `sdt diagnose` | Project-wide health report (lint + lineage + smell + cost + dim-modeling). | `--source`, `--min-severity`, `--category`, `--format` |
| `sdt review` | AI-assisted senior-DBA review of a project. *(Pro)* | `--project`, `--format` |
| `sdt cost-estimate` | Estimate warehouse cost of a planned migration. | `--script`, `--warehouse-size`, `--calibrate-from`, `--explain`, `--format`, `-o` |
| `sdt optimize` | Ranked heuristic optimization recommendations (clustering, narrowing, PKs, target-lag). | `--source`, `--severity`, `--explain`, `--format`, `-o` |
| `sdt docs` | Generate HTML schema docs from a project or pac. | `--source`, `-o`, `--title`, `--augment-comments` |
| `sdt erd` | Generate an ER diagram from a project or pac. | `--source`, `-o`, `--title` |
| `sdt profile` | Per-column profile statistics for a table. | `--rows`, `--live`, `--row-limit`, `--top-n`, `--format` |
| `sdt suggest-constraints` | Heuristic PK / UK / FK / CHECK candidates from a table profile. | `--profile`, `--parents`, `--include-low`, `--ai`, `--apply-to`, `--apply-min` |

---

## Data

Row-level data comparison, seeds, and the declarative test framework.

| Command | What it does | Key flags |
|---|---|---|
| `sdt data-compare` | Diff two tables by PK; emit the INSERT / UPDATE / DELETE script. | `--table`, `--key`, `--source`, `--target`, `--source-live`, `--target-live`, `--columns`, `--row-limit`, `--max-changes`, `--no-transaction`, `--format` |
| `sdt data-fit` | Read-only probes for narrowing-type changes (see Safety & rollback). *(Pro)* | `--source`, `--target`, `-o`, `--format` |
| `sdt seed` | Manage seed reference data. *(Pro)* | `list`, `render`, `apply`, `--project`, `--connection`, `--yes`, `--dry-run`, `--delete-not-in-source`, `--out` |
| `sdt test` | Declarative SQL test framework. *(Pro)* | `--project`, `--connection` |
| `sdt preview` | Read-only sample-data inspector (`SELECT * … LIMIT n`). | `--connection`, `--limit`, `--where`, `--sql-only`, `--format` |

---

## CI/CD

Gate pull requests, comment on diffs, and inspect resumable deploys. See [CI/CD](ci-cd.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt pr-comment` | Render a diff / lint / impact analysis as a PR Markdown comment. *(Team)* | `--base`, `--head`, `--format` |
| `sdt drift-gate` | Multi-region/replica drift gate; fail when any replica drifts past `--threshold`. *(Team)* | `--primary`, `--replicas`, `--threshold` |
| `sdt deploy-status` | Show or list resumable deploy checkpoints. | `[<deployId>]`, `--format`, `--state-dir`, `--remove` |
| `sdt advise-tests` | PR-time regression-test advisor (matches changed files to the feature catalog). | `--changed-files`, `--diff-summary`, `--existing-tests`, `--ai-max-spend`, `--format`, `-o` |
| `sdt review --senior-dba` | AI deploy-diff review; exits 2 on request-changes. *(Team)* | `--diff`, `--safety` |

---

## AI

BYO-key AI features, spend-capped locally. See [AI features](ai-features.md).

| Command | What it does | Key flags |
|---|---|---|
| `sdt ai` | Show + test the AI provider configuration. | `status`, `test --prompt`, `usage` |
| `sdt sketch` | Scaffold idiomatic Snowflake DDL from a prose description. *(Pro)* | `<kind>`, `--description`, `--target`, `--out` |
| `sdt suggest` | AI-assisted modeling suggestions (PKs, FKs, clustering keys). *(Pro)* | `--project`, `--kind`, `--min-confidence` |
| `sdt design` | AI-assisted generative schema design. *(Pro)* | `star`, `snowflake-schema`, `--description`, `--target`, `--fact`, `--dims`, `--refinement`, `--ai-max-spend` |
| `sdt safer-alternative` | Propose a safer DDL alternative for a flagged migration step. *(Pro)* | `--code`, `--fqn`, `--object-type`, `--reason`, `--gate`, `--sql`, `--format`, `--ai-max-spend` |
| `sdt pii` | Detect likely-PII columns and render MASKING POLICY DDL. | `scan`, `mask`, `--source`, `--ai`, `--unmasked-role`, `--verify-at-or-below`, `-o`, `--format` |

> [!NOTE]
> The `--explain` flag also adds AI narration to `compare`, `diagnose`, `review`, `impact`, `cost-estimate`, `lineage`, and `pr-comment`. *(Pro)*

---

## Migration & interop

Bring schemas in from other tools, export to other formats, and translate dialects.

| Command | What it does | Key flags |
|---|---|---|
| `sdt import` | Convert another tool's artifacts into a `.sdtproj`. | `--from schemachange\|snowddl\|sqlpackage-dacpac\|snowflake-account\|snowflake-dcm\|terraform-state\|sql-files`, `--source-path`, `--output`, `--connection` |
| `sdt migrate` | Migrate from another tool into SDT (v1: `from-dbt`). | `from-dbt`, `--output`, `--manifest`, `--identifier-case`, `--default-database`, `--default-schema` |
| `sdt migrate-platform` | Translate Snowflake `.sql` to another dialect (v1: `databricks`). | `--to`, `--output`, `--warn-on-unsupported` |
| `sdt export` | Export a `.sdtproj` to another tool's format (v1: Snowflake DCM Projects). | `dcm`, `--source`, `--output`, `--dcm-project` |
| `sdt schema` | Bidirectional schema interop (import / export / infer / validate-star). | `import-ddl\|import-dbml\|import-jsonschema\|import-dbt`, `export-dbml\|export-drawio\|export-jsonschema`, `infer-csv`, `infer-parquet`, `validate-star`, `suggest-snowflake-schema`, `--source`, `--out`, `--root`, `--name` |
| `sdt xcompare-ir` | Cross-platform IR compare (Snowflake ↔ Databricks); IR-level only. | `--source`, `--tier`, `--format`, `--allow-destructive`, `--allow-unmappable` |

---

## Utilities

Authoring helpers, observability, catalog cache, licensing, and meta commands.

### Authoring helpers

| Command | What it does | Key flags |
|---|---|---|
| `sdt suite` | Create or validate a multi-project Suite (`.sdtsuite`). *(Pro)* | `init`, `validate`, `--out`, `--name` |
| `sdt refactor` | Record a rename so compares treat it as a rename, not drop + add. | `rename`, `list`, `--object`, `--new-name` |
| `sdt template` | Schema cookbook generator (SCD1–6, star, fact, time-series, audit, …). | `<kind>`, `--dims`, `--source`, `--out` |
| `sdt generate` | Code-gen for ingest + load scaffolds. | `notebook`, `scd`, `star-pipeline`, `--table`, `--natural-key`, `--type`, `--from-json`, `--from-design`, `--format`, `--out` |
| `sdt snippets` | Browse the built-in SQL snippet catalog. | `list --category`, `show --format` |
| `sdt branch` | Warehouse-native branching via zero-copy clones. *(Team)* | `create`, `drop`, `list`, `--from`, `--connection`, `--pattern`, `-o` |
| `sdt anonymize` | Sanitized copy of a `.sdtproj` for bug reports / demos. | `--project`, `--output` |

### Observability & history

| Command | What it does | Key flags |
|---|---|---|
| `sdt history` | Show recent deploys from the local history store. | `--limit` |
| `sdt bisect` | Find the commit where an FQN first changed (pure-git). | `<fqn>`, `--good`, `--bad`, `--object-type`, `--format` |
| `sdt changelog` | Generate a Markdown CHANGELOG from Conventional Commits. | `--from`, `--to`, `--version`, `-o`, `--append` |
| `sdt audit-log` | Export deploy + compare records to an enterprise sink. | `emit`, `--sink file\|syslog\|splunk\|datadog\|otlp`, `--workspace`, `-o`, `--dry-run` |
| `sdt telemetry` | Manage the local telemetry opt-in (also governs error reporting). | `status`, `on`, `off` |
| `sdt query-log` | Browse the local query log written by the VS Code query window. | `list`, `show`, `add`, `clear`, `--profile`, `--limit`, `--yes` |
| `sdt error-lookup` | Look up a deploy/adapter failure in the known-error catalog. | `--code`, `--fingerprint`, `--message`, `--list`, `--format` |

### Catalog cache & explorer

| Command | What it does | Key flags |
|---|---|---|
| `sdt catalog` | Manage the per-connection catalog cache (feeds Object Explorer + intellisense). | `refresh`, `show`, `clear`, `--connection`, `--databases`, `--concurrency`, `--include-builtins`, `--no-fingerprint-skip`, `--json` |
| `sdt explorer` | ASCII tree of the live db/schema/object hierarchy (reads the cache). | `--connection`, `--filter`, `--depth`, `--json` |
| `sdt compare-profiles` | Manage saved compare profiles (source → target + mapping rules). *(Pro)* | `list`, `show`, `save`, `remove`, `preview`, `--json-file`, `--json`, `--examples` |
| `sdt search` **(beta)** | Find objects matching a pattern across profiles, flagging presence drift. | `--profiles`, `--exact`, `--type`, `--no-drift`, `--format` |

### Licensing, meta & integration

| Command | What it does | Key flags |
|---|---|---|
| `sdt license` | Activate / show / deactivate a license. | `activate --key`, `status`, `deactivate` |
| `sdt trial` | Manage the local 30-day Pro trial (no account needed). | `start`, `status` |
| `sdt pilot` | Join or check the pilot program (90-day Pro access). | `register --email`, `status` |
| `sdt features` | Inspect the visible-but-locked feature catalog. | `list`, `show`, `upgrade` |
| `sdt find` | Search the feature catalog by keyword. | `--limit`, `--tier`, `--shipped`, `--ai`, `--ai-threshold` |
| `sdt explain` | Search the options catalog for a flag / config key / behavior. | `--tag`, `--safety`, `--limit`, `--list` |
| `sdt discover` | Personalized feature suggestions from your usage history. | `--reset` |
| `sdt hosts` | Inspect the host-adapter catalog (every CLI/IDE/CI surface). | `list`, `show <id>` |
| `sdt mcp` | Run the MCP server (stdio JSON-RPC) so AI agents can invoke SDT as a tool. | — |
| `sdt completion` | Emit a shell-completion script. | `bash\|fish\|zsh\|powershell` |
| `sdt feedback` | Send a feedback message to the support inbox. | `"<message>"`, `--category` |
| `sdt savings` | Estimated AI-token savings from deterministic surfaces. | `--top`, `--format`, `--reset` |
| `sdt bookmarks` | Saved pointers to queries, files, or sections. | `add`, `remove`, `get`, `list`, `--type`, `--target`, `--label`, `--format` |
| `sdt perf` | CLI performance diagnostics (cold-start latency). | `cold-start`, `--json`, `--budget` |
| `sdt watch` | Diff a Snowflake release-notes file against a snapshot. | `bulletin`, `--current`, `--prior`, `--save-snapshot`, `--format` |
| `sdt --version` | Print the installed version. | — |

---

**Next:** [VS Code extension reference](vscode-extension.md) · **Up:** [Documentation home](README.md)
