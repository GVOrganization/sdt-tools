# 🚦 The safety classifier

> Understand exactly how SDT decides a change is safe, expensive, or dangerous — and which opt-in unlocks each blocked operation.

**On this page:** [What it does](#what-it-does) · [The categories](#the-categories) · [Opt-in gates](#opt-in-gates) · [Where findings appear](#where-findings-appear) · [Reversibility grouping](#reversibility-grouping) · [Inspecting finding codes](#inspecting-finding-codes)

---

## What it does

The safety classifier is the heart of SDT — Snowflake Data Tools. Every change SDT plans is run through it before any SQL executes. It reads two inputs — the compare result and your resolved deploy options — and refuses to deploy unsafe operations unless you have explicitly opted in.

This is the *no silent destruction* guarantee: a destructive operation is never executed by accident. It is either unlocked by you, or it stays commented out and blocked.

---

## The categories

SDT classifies every change into one of four categories.

| Category | What it means | Example operations | What unlocks it |
|---|---|---|---|
| `UNRECOVERABLE` | State that cannot be reconstructed even from a Snowflake clone. | Dropping streams (cursor), pipes (load history), tasks (scheduling state), warehouses (running queries), integrations. | `allowUnrecoverableDrop: true` |
| `DESTRUCTIVE` | Drops persistent rows or narrows column semantics. | `DROP TABLE`, column drops, type-narrowing (`VARCHAR(255)` → `VARCHAR(50)`), `BLOCKED`-rebuild changes. | `allowDropTable`, `allowDropColumn`, `allowNarrowingTypes`, `allowTableRebuild` (per operation); blocked while `blockOnPossibleDataLoss: true` |
| `EXPENSIVE` | Full rewrites, materializations, DAG suspensions — costly but not data-losing. | Clustering changes, dynamic-table query swaps, materialized-view rebuilds. | Allowed by default; surfaced in the safety report. |
| `WARNING` | Adds of state-bearing objects, and drops skipped by `doNotDropObjectTypes`. | New streams / tasks / pipes (they start running immediately on deploy). | Allowed by default; `treatWarningsAsErrors: true` blocks. |

> [!NOTE]
> Why streams, pipes, and tasks are `UNRECOVERABLE`: a stream holds an offset/cursor over its source, a pipe holds 14-day load history Snowflake uses to de-dup, and a task holds scheduling state. Dropping any of these loses state that no clone can restore — often invisibly, until a downstream consumer desyncs.

---

## Opt-in gates

A blocked operation reports which gate it needs. Set the gate as a `deployOptions` key for `sdt publish`, or pass the matching flag to `sdt purge`.

| Gate / option | What it unlocks |
|---|---|
| `allowDropTable` | `DROP TABLE`. |
| `allowDropColumn` | Column drops (`ALTER TABLE … DROP COLUMN`). |
| `allowNarrowingTypes` | Narrowing a column type (e.g. `VARCHAR(255)` → `VARCHAR(50)`). |
| `allowTableRebuild` | Rebuilds where the strategy is `BLOCKED` (clustering changes, etc.). |
| `allowUnrecoverableDrop` | Drops of stream / pipe / task / similar unrecoverable types. |
| `allowExternalLanguageChanges` | Java / Python UDF and procedure changes (`true` by default). |

Two top-level options control the overall posture:

| Option | Default | Effect |
|---|---|---|
| `blockOnPossibleDataLoss` | `true` | Refuse any deploy with a destructive or unrecoverable change. |
| `treatWarningsAsErrors` | `false` | Promote every WARNING finding to a blocker (CI-strict mode). |

These confirmation gates layer on top:

| Gate | Satisfied by |
|---|---|
| `REQUIRE_YES_FLAG` | `--yes` on the CLI. |
| `REQUIRE_CONFIRM_PRODUCTION` | `--confirm-production` (required for prod-named profiles). |

`sdt purge` exposes the per-operation allowances as flags directly: `--allow-drop-table`, `--allow-unrecoverable-drop`, `--allow-narrowing-types`, `--allow-table-rebuild`.

> [!WARNING]
> Unlocking a gate does not make the operation safe — it only tells SDT you accept the consequence. A `DROP TABLE` with `allowDropTable: true` still destroys every row in the table.

---

## Where findings appear

The same classification surfaces in three places, so the verdict is consistent whether you are reviewing a diff, reading a script, or gating CI.

**In compare results** — `sdt compare` prints findings inline, color-coded by category: UNRECOVERABLE in bold red, DESTRUCTIVE in red, EXPENSIVE in yellow, WARNING in cyan, SAFE/OK in green.

**In migration scripts** — any non-`SAFE` drop is emitted as a commented-out statement with a `-- WARNING:` note, and blocked operations carry a `-- BLOCKED:` line naming the gate that would enable them:

```sql
-- WARNING: dropping a STREAM is unrecoverable — the cursor/offset is lost.
-- BLOCKED: set allowUnrecoverableDrop=true to enable.
-- DROP STREAM ANALYTICS.PUBLIC.ORDERS_STREAM;
```

**In CLI exit codes** — for CI gating:

| Code | Meaning |
|---|---|
| 0 | Success |
| 3 | Compare diff found (flag-controlled) |
| 4 | Destructive changes detected and not approved |
| 5 | Drift detected (`sdt drift` in CI mode) |

---

## Reversibility grouping

Category answers *how dangerous?* Reversibility answers a different question: *if this goes wrong, can I undo it?* The two axes don't always agree — a DESTRUCTIVE change is often reversible (restore from a clone), and a WARNING change can be irreversible (adding a `NOT NULL` column to a non-empty table).

Both `sdt compare` and `sdt publish --dry-run` print a reversibility summary after the per-object diff:

| Bucket | Glyph | Contents |
|---|---|---|
| Unrecoverable | 🛑 | UNRECOVERABLE findings — `DROP TABLE` without retention, `DROP DATABASE`, etc. |
| Data-impacting | ⚠ | DESTRUCTIVE + EXPENSIVE findings — can lose or rewrite data. |
| Reversible | ℹ | WARNING findings — visible but recoverable. |

```text
🛑 UNRECOVERABLE (1)
  • DROP TABLE ANALYTICS.PUBLIC.AUDIT_LOG (no retention policy)
    → cannot recover without external backup

⚠ DATA-IMPACTING (2)
  • ALTER TABLE ANALYTICS.PUBLIC.ORDERS DROP COLUMN MEMO
    → column data lost; revert requires re-extract from history
  • REBUILD ANALYTICS.PUBLIC.CUSTOMER_AGG (clustering key changed)
    → 4.3M rows × 12 GB warehouse-time cost

ℹ REVERSIBLE (1)
  • ALTER TABLE ANALYTICS.PUBLIC.ORDERS ADD COLUMN STATUS VARCHAR(20)
    → safe; can be dropped if needed
```

---

## Inspecting finding codes

Each finding carries a stable code. Browse the catalogue or get the deep "why this is dangerous" write-up for one code:

```sh
# List every finding code in a category
sdt safety list --category unrecoverable

# Explain a single code in full
sdt safety explain DROP_UNRECOVERABLE --format markdown
```

`list` groups codes by category (`unrecoverable | destructive | expensive | warning`); `explain <code>` exits non-zero on an unknown code.

---

**Next:** [Configuration reference](configuration.md) · **Up:** [Documentation home](README.md)
