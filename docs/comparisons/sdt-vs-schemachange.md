# SDT vs schemachange

Both deploy schema changes to Snowflake from files in git. They disagree on one fundamental thing: **what the source of truth is**.

## TL;DR

- Pick **schemachange** if: you want a minimal, open-source, migration-numbered runner and you're comfortable hand-writing every forward change (and its rollback) yourself.
- Pick **SDT** if: you want the *desired state* in git and a compare engine to compute the migrations — plus safety classification, drift detection, and rollback manifests.

## The core difference

**schemachange is migration-based.** You write `V1.1.1__add_orders.sql`, `V1.1.2__add_column.sql`, … and schemachange applies the ones the target hasn't seen (tracked in a change-history table). The *history of edits* is the source of truth. Nobody can answer "what should production look like?" without replaying the chain in their head.

**SDT is state-based.** Each object is one `.sql` file describing what it *should* look like. `sdt compare` diffs your project against the live account (or a built artifact) and generates the migration script. The *desired state* is the source of truth; migrations are computed, not authored.

## Honest feature comparison

| Capability | schemachange | SDT |
|---|---|---|
| Open source | ✅ Apache-2.0, Snowflake-Labs | ❌ closed-source (free core tier; artifact licensed Apache-2.0) |
| Simplicity (one Python script, no model) | ✅ | ❌ — full object model, bigger surface |
| Jinja templating in scripts | ✅ | ❌ (variables + per-env profiles instead) |
| Battle-tested since 2019 | ✅ | ❌ public beta (2026) |
| Compare project ↔ live account | ❌ | ✅ any direction, incl. offline artifact |
| Hand-writing migrations required | every change | never — generated |
| Safety classification on changes | ❌ | ✅ `SAFE` / `EXPENSIVE` / `DESTRUCTIVE` / `UNRECOVERABLE`, destructive ops refuse without an explicit gate |
| Drift detection (prod vs git) | ❌ | ✅ `sdt drift` |
| Rollback story | write your own undo scripts | ✅ per-deploy manifest revert + optional zero-copy clone |
| Extract existing account to files | ❌ | ✅ `sdt extract` |
| Rename without DROP+CREATE | manual | ✅ refactor log |
| VS Code integration | ❌ | ✅ extension (compare UI, object explorer, lineage) |
| Runs with no warehouse credentials | ❌ (needs a connection) | ✅ build/compare/lint fully offline |

## When to pick which

1. **Do you have hundreds of historical `V__` scripts and a team fluent in them?** Staying on schemachange is reasonable — the migration chain *is* your institutional memory.
2. **Do you need to answer "what's different between git and prod right now"?** schemachange structurally can't; that's a compare engine. SDT.
3. **Is open source a hard requirement?** schemachange (or see [Atlas](https://atlasgo.io)).
4. **Has a bad `DROP`/`ALTER` ever cost you data?** The safety classifier is the reason SDT exists.

## Migrating from schemachange

The importer walks your scripts in version order and computes the final declarative state:

```sh
sdt import --from schemachange --source-path ./migrations --output ./MyProject
```

Scripts containing bulk DML or unclassifiable operations are surfaced as warnings with file + line, so you can fold them in by hand. Your schemachange history table is untouched — you can run both side by side while you evaluate. See [Migrating from other tools](../migrating.md).

---

*Independent comparison written by the SDT maintainer — corrections welcome via [issues](https://github.com/GVOrganization/sdt-tools/issues). schemachange is a Snowflake-Labs community project.*
