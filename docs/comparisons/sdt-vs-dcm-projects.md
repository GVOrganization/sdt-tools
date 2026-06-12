# SDT vs Snowflake DCM Projects

Snowflake's native **DCM (Database Change Management) Projects** brought declarative, state-based schema deployment into the platform itself. It's the same model SDT is built on — which says the model is right. This page is where they actually differ today.

## TL;DR

- Pick **DCM Projects** if: you want a zero-install, Snowflake-supported way to apply declarative DDL from inside the warehouse, and Preview-grade tooling is acceptable.
- Pick **SDT** if: you need the compare itself to be trustworthy, you want safety classification on every change, or you need any of it to run outside a warehouse (CI, pre-commit, laptop, air-gapped).
- Plenty of teams will run **both**: DCM for in-warehouse orchestration, SDT for the compare + safety layer on top. SDT reads DCM projects (`sdt import --from snowflake-dcm`).

## Honest feature comparison

| Capability | DCM Projects | SDT |
|---|---|---|
| Native, zero install, Snowflake-supported | ✅ | ❌ third-party |
| Price | included with Snowflake | free core tier; paid tiers for Pro features |
| Maturity | Preview | public beta |
| State-based declarative model | ✅ | ✅ |
| Changeset / diff fidelity | explicitly *not* guaranteed in Preview | the compare engine **is** the product |
| Safety classification | ❌ (execute with warnings) | ✅ four tiers + reversibility analysis; destructive ops refuse without explicit gates |
| Runs offline (no warehouse compute) | ❌ — build/plan/apply run in-warehouse | ✅ build/compare/classify/lint need no credentials |
| Entity scale | 20k-entity cap | no architectural cap |
| Masking / row-access policy attachment in column definitions | ❌ | ✅ |
| Rename detection / refactor log | ❌ | ✅ |
| Drift detection as a CI gate | in-warehouse only | ✅ offline `sdt drift` |
| Rollback | re-apply prior state | per-deploy manifest revert + optional zero-copy clone |
| Import from schemachange / dbt / DACPAC / Terraform state | ❌ | ✅ |
| VS Code integration | ❌ | ✅ |
| Works for Databricks too | ❌ Snowflake-only | sibling tool ([DDT](https://github.com/GVOrganization/ddt-tools)) shares the CLI surface |

## The two differences that matter most

1. **Where it runs.** Every DCM operation consumes warehouse compute and needs a connection. SDT's build/compare/classify pipeline is fully offline — you can block a destructive change in a pre-commit hook on a plane. If your goal is "no unsafe DDL reaches the account," the gate has to live *before* the account.
2. **Whether the diff is the product or a byproduct.** DCM's changeset exists to drive its execute step and is explicitly not fidelity-guaranteed in Preview. SDT's compare output is the thing you review, gate on, and attach to the PR — fidelity is the entire value proposition, and where the engine can't guarantee a faithful diff for a long-tail object type, it says so rather than guessing.

## Using them together

```sh
# Keep authoring in DCM format; add SDT's compare + safety as a CI gate
sdt import --from snowflake-dcm --source-path ./dcm_project --output ./Project
sdt compare --project ./Project --target prod   # classified diff, offline
```

If Snowflake graduates DCM with fidelity guarantees, safety classification, and offline execution, this page will say so — they validate the category either way.

---

*Independent comparison written by the SDT maintainer — corrections welcome via [issues](https://github.com/GVOrganization/sdt-tools/issues). Not affiliated with or endorsed by Snowflake Inc. Claims reflect DCM Projects as of mid-2026 (Preview); verify against current Snowflake docs.*
