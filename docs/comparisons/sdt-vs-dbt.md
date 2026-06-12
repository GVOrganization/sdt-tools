# SDT vs dbt — complement, not competitor

This is the comparison page that isn't one. dbt and SDT manage **different layers** of the same warehouse, and most teams that adopt SDT keep dbt exactly as it is.

## TL;DR

- Use **dbt** for: transformations — models, tests, materializations, documentation of the modeled layer.
- Use **SDT** for: everything dbt deliberately doesn't model — tables created outside dbt, streams, tasks, pipes, stages, file formats, procedures, UDFs, masking/row-access policies, warehouses — with compare, drift detection, and safety classification.
- They meet at the boundary: SDT imports your dbt `manifest.json` so the modeled layer and the DDL layer live in one compared, classified view.

## The layer split

dbt's design philosophy is explicit: it owns `SELECT`-derived objects. Ask a dbt shop "how do you ship a new source table, a stream on it, and the task that consumes the stream?" and the answer is some mix of hand-run DDL, a `run-operation` macro escape hatch, or a wiki page — the exact ad-hoc workflow dbt eliminated for models.

| Layer | Owner |
|---|---|
| Sources, raw/landing tables | **SDT** (dbt only *declares* sources; it doesn't create or alter them) |
| Models (views/tables from `SELECT`) | **dbt** |
| Streams, tasks, pipes, stages, file formats | **SDT** (dbt has no story) |
| Procedures, UDFs | **SDT** (macro escape hatches aside) |
| Masking / row-access policies, governance objects | **SDT** |
| Tests on data values | **dbt** |
| Safety classification of DDL changes | **SDT** |
| Schema drift vs git | **SDT** (`sdt drift`) — dbt notices drift only when a run breaks |

## What about dbt Fusion?

The Fusion engine gives dbt real SQL comprehension and column-level lineage — a genuine leap for the modeled layer. It doesn't change the boundary: Fusion still models *models*. Non-model DDL — the stream/task/policy/raw-table layer — remains outside dbt's design scope, Fusion or not.

## Honest limits of the complement story

- If your warehouse is **100% dbt-modeled** (every object a model, sources managed elsewhere), SDT adds little — you don't have the layer it manages.
- SDT does **no transformation work at all** — no Jinja, no incremental materialization, no data tests. It will never replace dbt and doesn't try.
- Two tools is two tools. The integration reduces the seam but doesn't remove it.

## Using them together

```sh
# Seed an SDT project from your dbt project's manifest —
# models and sources arrive as declared objects:
sdt import --from dbt --source-path ./target/manifest.json --output ./Warehouse

# dbt keeps owning the models; SDT compares + classifies everything,
# and flags drift between either layer and the live account:
sdt compare --project ./Warehouse --target prod
```

A useful CI pattern: `dbt build` gates the transformation layer, `sdt drift --target prod` gates the DDL layer — both fail the pipeline, each for the layer it owns. See [CI/CD integration](../ci-cd.md).

---

*Independent comparison written by the SDT maintainer — corrections welcome via [issues](https://github.com/GVOrganization/sdt-tools/issues). "dbt" is a trademark of dbt Labs, Inc.*
