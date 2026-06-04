# 🔍 Schema compare

> Diff any two of your project, a built artifact, or a live Snowflake account — and see exactly what a deploy would change before it runs.

**On this page:** [The compare model](#the-compare-model) · [Reading results](#reading-results) · [VS Code flow](#vs-code-flow) · [CLI flow](#cli-flow) · [Compare audit trail](#compare-audit-trail) · [Drift check](#drift-check)

---

## The compare model

![Schema compare demo](../assets/demo-compare.gif)

SDT's compare engine diffs two **sources** in any direction. A source is one of:

| Source | What it is |
|---|---|
| Project (`.sdtproj`) | Your working tree of `.sql` files — the desired state. |
| Build artifact (`.sdtpac`) | A compiled, frozen snapshot from `sdt build`. |
| Live account (`snowflake://<profile>[/<db>[/<schema>]]`) | A connected Snowflake account, read live. |

Any combination works. The everyday ones:

- **Project ↔ live** — does the account match what I authored?
- **Project ↔ pac** — what changed since the last build?
- **Pac ↔ pac** — audit two tagged releases against each other.

The same engine powers `sdt publish` and `sdt drift`, so the diff you review in compare is the diff a deploy acts on.

---

## Reading results

Every object is classified by **status**:

| Status | Meaning |
|---|---|
| Added | Present in the source, absent from the target. |
| Removed | Present in the target, absent from the source. |
| Modified | Present in both, but differs — drill into field-level changes. |
| Unchanged | Identical on both sides. |

On top of status, every change carries a **safety category** so you know which diffs are dangerous before you act on them:

| Category | What it signals |
|---|---|
| UNRECOVERABLE | Cannot be undone — data or an object is lost with no rollback. |
| DESTRUCTIVE | Drops or rebuilds that lose data unless explicitly allowed. |
| EXPENSIVE | Safe, but costly — a full table rebuild or large rewrite. |
| WARNING | Worth a look, but not destructive. |

> [!WARNING]
> Always read the safety report before deploying. UNRECOVERABLE and DESTRUCTIVE findings are gated behind explicit opt-ins for a reason.

See [Safety classifier](safety-classifier.md) for the full category breakdown and the stable finding codes, and [Safe deploy](safe-deploy.md) for how the gates turn into deploy options.

---

## VS Code flow

The **Schema Compare** webview is SDT's flagship surface. Run **SDT: Schema Compare** from the Command Palette (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>) and pick the two sources.

The view shows:

- A tree of every object grouped by type, with **added / removed / modified / unchanged** status badges.
- Per-object, field-level drill-down for modified rows.
- Safety findings inline — every diff classified UNRECOVERABLE / DESTRUCTIVE / EXPENSIVE / WARNING.
- Include/exclude checkboxes per row — your selection narrows the generated script.
- A **Generate Script** action that emits the migration SQL into an editor, with `-- WARNING:` comments around any non-SAFE operation.

> [!TIP]
> Use the include/exclude checkboxes to carve a large diff into a focused, reviewable migration before you generate the script.

---

## CLI flow

`sdt compare` diffs two sources and prints the result. The source order is `<source> <target>`.

```sh
# Project vs live account
sdt compare ./MyProject.sdtproj 'snowflake://prod/MY_DB'

# Pac vs pac — audit two builds, as Markdown
sdt compare ./bin/v1.sdtpac ./bin/v2.sdtpac --format markdown

# Project vs live with AI narration (Pro)
sdt compare ./MyProject.sdtproj 'snowflake://prod' --explain
```

| Flag | What it does | Notes |
|---|---|---|
| `--format <fmt>` | Output format: `summary`, `json`, `markdown`. | Defaults to `summary` for humans. |
| `--ignore-case` | Treat unquoted identifiers case-insensitively. | Useful when one side quotes identifiers and the other doesn't. |
| `--explain` | Add AI narration of the diff. | Pro tier; requires a configured AI provider. |
| `--color <when>` | `always` / `never` / `auto`. | `auto` honors the TTY and the `NO_COLOR` env var. |
| `--map <src=tgt>` | Rewrite source-side names so differently-named schemas compare semantically. | Repeatable; `--map-file <path>` reads a list. |
| `--type-safe` | Run impact analysis after the compare and gate the exit code. | Pair with `--break-on`. |
| `--break-on <sev>` | Threshold for `--type-safe`: `error` (default) or `warning`. | `warning` is strict CI mode. |
| `--no-history` | Skip writing the compare audit record. | For read-only CI environments. |

Severity colors: UNRECOVERABLE renders bold red, DESTRUCTIVE red, EXPENSIVE yellow, WARNING cyan, SAFE/OK green.

### Exit codes

`sdt compare` returns a meaningful exit code so it can gate CI:

| Code | Meaning |
|---|---|
| 0 | Success — no gate tripped. |
| 1 | User error (bad args, missing file, syntax error). |
| 2 | System error, **or** a `--type-safe` gate tripped at its `--break-on` threshold. |
| 3 | Compare diff found (when used to detect drift in CI). |
| 6 | A locked feature was attempted without the required license tier. |

```sh
# CI gate: fail the build when a change ripples a type error
sdt compare ./MyProject.sdtproj ./bin/last.sdtpac --type-safe --format json

# Strict mode + impact file the editor reads back as squiggles
sdt compare ./MyProject.sdtproj ./bin/last.sdtpac \
  --type-safe --break-on warning --write-impact
```

### Comparing differently-named environments

When dev and prod use different database or schema names, every object reads as added/removed. Map the names so the compare is semantic:

```sh
# Compare a dev pac against prod naming without false diffs
sdt compare ./dev.sdtpac ./prod.sdtpac --map DEV_DB=PROD_DB
```

`--map` is repeatable, and `--map-file <path>` loads a list. The same mapping works on `sdt script`, `sdt drift`, and `sdt publish`.

---

## Compare audit trail

Every compare run records a history entry under `.sdt/history/compare/` next to the source — capturing the outcome (`NO_CHANGES` / `CHANGES_FOUND` / `BLOCKED`), the source and target labels, and a diff summary. Pass `--no-history` to skip the write in read-only environments.

These records export alongside deploy history via `sdt audit-log emit`, giving you one ordered stream of "what was compared and what was deployed."

---

## Drift check

Compare answers "how do these two sources differ?" **Drift** answers a narrower, recurring question: "has the live account drifted from what my project expects?" It's compare in report-only mode, purpose-built for a scheduled or pre-deploy check.

```sh
# Report drift between the project and the live account
sdt drift --project ./MyProject.sdtproj --connection prod
```

For multi-region or replica fleets, `sdt drift-gate` compares each replica against a primary and **fails** the gate when any replica drifts beyond a threshold — drop it into CI:

```sh
sdt drift-gate \
  --primary prod-us \
  --replicas prod-eu,prod-apac \
  --threshold 0
```

When `sdt drift` runs in CI mode, drift detection returns exit code 5, so a pipeline can act on it automatically.

See [CI/CD](ci-cd.md) for wiring drift and type-safe gates into a pipeline.

---

**Next:** [Safe deploy](safe-deploy.md) · **Up:** [Documentation home](README.md)
