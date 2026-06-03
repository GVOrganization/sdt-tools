# 🛡️ Safe deploy

> Build your project into an artifact, preview the exact SQL, and apply it to Snowflake with destructive changes blocked unless you opt in.

**On this page:** [The deploy loop](#the-deploy-loop) · [Dry-run, then apply](#dry-run-then-apply) · [How blocked changes look](#how-blocked-changes-look) · [Opt-in gates](#opt-in-gates) · [Production safeguards](#production-safeguards) · [Rollback](#rollback)

---

## The deploy loop

![SDT safe deploy demo](../assets/demo-deploy.gif)

SDT — Snowflake Data Tools deploys in four steps. Each step is its own command, so you always see what will happen before anything touches the account.

1. **Build** — compile the `.sdtproj` into a `.sdtpac` artifact.
2. **Dry-run** — generate the migration SQL and the safety assessment, execute nothing.
3. **Apply** — run the migration against the connection, capturing a manifest for revert.
4. **Roll back** — if needed, revert from the manifest or restore from a pre-deploy clone.

> [!IMPORTANT]
> SDT is state-based: you describe the desired state in `.sql` files and SDT computes the deltas. Re-running `sdt publish` against an up-to-date target is a no-op.

---

## Dry-run, then apply

Always dry-run first. The dry-run prints the migration SQL exactly as it will run.

```sh
# 1. Build the project into a .sdtpac artifact
sdt build --project ./MyProject.sdtproj --out ./bin/MyProject.sdtpac

# 2. Dry-run: print the migration SQL, execute nothing
sdt publish --pac ./bin/MyProject.sdtpac --connection prod --dry-run

# 3. Apply for real, capturing a manifest you can revert from later
sdt publish --pac ./bin/MyProject.sdtpac --connection prod \
            --apply --yes \
            --manifest ./deploy-2026-05-14.json
```

| Flag | What it does |
|---|---|
| `--dry-run` | Print the migration SQL; don't execute. |
| `--apply` | Execute the migration against the connection. |
| `--yes` | Confirm a destructive or non-interactive deploy (required when not on a TTY). |
| `--manifest <path>` | Capture a deploy manifest so the deploy can be reverted. |

> [!TIP]
> To hand the SQL to a DBA for review without contacting the account at all, use `sdt script` — it is always offline and never touches a live Snowflake account.

---

## How blocked changes look

SDT never silently destroys data. When the migration contains a destructive or unrecoverable operation that you have not unlocked, SDT emits the statement **commented out** with a `-- WARNING:` note explaining why, and the deploy is blocked.

```sql
-- WARNING: DROP TABLE is destructive — column/row data is lost.
-- BLOCKED: set allowDropTable=true (deployOptions) or pass --allow-drop-table to enable.
-- DROP TABLE ANALYTICS.PUBLIC.ORDERS_OLD;
```

The dry-run output ends with a safety report grouped by reversibility, so you can see at a glance what you would need to roll back:

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

For the full category model and what each category means, see [The safety classifier](safety-classifier.md).

---

## Opt-in gates

Destructive operations stay blocked until you explicitly allow them. Set the gate in your project's `deployOptions` (per-environment, under `deploymentProfiles.<env>.deployOptions`), or pass the matching CLI flag.

| You want to allow | `deployOptions` key | `sdt publish` |
|---|---|---|
| `DROP TABLE` | `allowDropTable: true` | (set in deployOptions) |
| Column drops | `allowDropColumn: true` | (set in deployOptions) |
| Narrowing a column type | `allowNarrowingTypes: true` | (set in deployOptions) |
| Full-object rebuilds (clustering changes, etc.) | `allowTableRebuild: true` | (set in deployOptions) |
| Dropping streams / pipes / tasks | `allowUnrecoverableDrop: true` | (set in deployOptions) |

```jsonc
// In MyProject.sdtproj — loosen only the dev profile
"deploymentProfiles": {
  "dev": {
    "connection": "snowflake-dev",
    "deployOptions": {
      "deployment": {
        "blockOnPossibleDataLoss": false,
        "allowDropTable": true,
        "allowTableRebuild": true
      }
    }
  }
}
```

> [!WARNING]
> `blockOnPossibleDataLoss` is `true` by default and refuses any deploy with a destructive or unrecoverable change. Turning it off in a production profile removes the single biggest guard SDT gives you. Loosen gates per environment, never globally.

The full set of gates and exactly what each unlocks is documented in [The safety classifier](safety-classifier.md) and the [Configuration reference](configuration.md).

---

## Production safeguards

When the active profile name matches `prod`, `prd`, `production`, or `live` (or you pass `--production`), SDT raises the bar:

- Both `--yes` **and** `--confirm-production` are required for any destructive operation.
- A pre-deploy clone is offered automatically; set `preDeployClone: true` to make it the default.

```sh
# Production deploy — double opt-in required
sdt publish --pac ./bin/MyProject.sdtpac --connection prod \
            --apply --yes \
            --manifest ./deploy-2026-05-14.json \
            --confirm-production
```

> [!WARNING]
> `--confirm-production` is a deliberate second confirmation, not a formality. SDT requires it precisely because the target is a profile that looks like production.

You can also set `treatWarningsAsErrors: true` to promote every WARNING finding to a blocker — a good CI-strict default.

---

## Rollback

SDT gives you three recovery levers.

### Zero-copy clone + SWAP

Snowflake's zero-copy clone makes a safe rollback point essentially free. With `preDeployClone: true`, the migration starts by cloning the target and prints the rollback `ALTER … SWAP WITH` command on success.

```sql
-- The pre-deploy clone (cheap; no data is copied)
CREATE SCHEMA ANALYTICS_BACKUP_20260512 CLONE ANALYTICS;
```

> [!NOTE]
> Auto-clone-on-apply with the printed SWAP rollback is planned for a future release. Today you can take the clone yourself, or rely on the deploy manifest and pre-deploy snapshots below.

### Revert from a manifest

Every `sdt publish --apply --manifest <path>` captures the forward and reverse SQL. `sdt revert` replays each step's reverse statement against the account in reverse order.

```sh
# Preview what the revert would run
sdt revert --manifest ./deploy-2026-05-14.json --connection prod --yes --dry-run

# Apply the revert
sdt revert --manifest ./deploy-2026-05-14.json --connection prod --yes
```

> [!WARNING]
> Revert is itself destructive — `--yes` is mandatory. Steps with no captured reverse SQL (DROPs, for example) are inherently irreversible and are skipped with a warning; the exit code is non-zero when any step could not be reversed.

### Restore from a pre-deploy snapshot

`sdt publish --apply` writes a pre-deploy snapshot batch you can restore from with a single command — this is the command SDT prints when a deploy fails.

```sh
# Restore from the snapshot batch (dry-run by default)
sdt publish --restore-from-snapshot snap_lqz3k_4f8a2c --connection prod \
            --apply --yes --confirm-production
```

Manage the snapshot registry with `sdt snapshot list / show / prune`.

---

**Next:** [The safety classifier](safety-classifier.md) · **Up:** [Documentation home](README.md)
