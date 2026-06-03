# 🚚 Migrating from other tools

> Convert an existing schema-management setup — schemachange, SnowDDL, a dacpac, Terraform state, dbt, or a folder of raw SQL — into an SDT project.

**On this page:** [How import works](#how-import-works) · [schemachange](#schemachange) · [SnowDDL](#snowddl) · [SqlPackage / dacpac](#sqlpackage--dacpac) · [Terraform state](#terraform-state) · [Live account](#live-account) · [Flat SQL files](#flat-sql-files) · [dbt](#dbt) · [Run both in parallel](#run-both-in-parallel)

---

## How import works

`sdt import --from <tool>` reads another tool's artifacts and writes a fresh `.sdtproj` with each Snowflake object as its own `.sql` file. **Importers never modify the source** — they only read it and write a new project tree.

```bash
sdt import --from <tool> --source-path <path> --output ./my-project
```

| Flag | What it does |
|---|---|
| `--from <tool>` | Source format: `schemachange`, `snowddl`, `sqlpackage-dacpac`, `snowflake-account`, `snowflake-dcm`, `terraform-state`, `sql-files` |
| `--source-path <path>` | Path to the source artifacts (required) |
| `--output <dir>` | Where to write the new `.sdtproj` (required) |
| `--connection <profile>` | Required only for `snowflake-account` |

Once imported, you author declaratively going forward — SDT computes deltas from desired state rather than tracking numbered migration scripts.

---

## schemachange

[schemachange](https://github.com/Snowflake-Labs/schemachange) is migration-based; SDT is state-based. The importer replays your history into a desired-state project.

| schemachange concept | SDT concept |
|---|---|
| `V*.sql` versioned scripts | Replayed in order into an in-memory model |
| `R__*.sql` repeatable scripts | Applied after the versioned scripts |
| Resulting object state | One declarative `.sql` per object |
| Non-declarable DML / ad-hoc calls | Preserved as `migrations/V### …` files |

```bash
sdt import --from schemachange --source-path ./migrations --output ./my-project
```

**After import:** you get a hybrid model — declarative state for every object, plus the numbered `migrations/` files that preserve your historic order. Start authoring declaratively; the migrations stay as a record.

---

## SnowDDL

[SnowDDL](https://github.com/littleK0i/SnowDDL) is already declarative, so the import is a fairly direct translation — both tools target desired state.

```bash
sdt import --from snowddl --source-path ./snowddl-config --output ./my-project
```

**After import:** review the generated objects, then build and compare against your account to confirm parity before deploying.

---

## SqlPackage / dacpac

For shops moving off SQL Server. The importer reads a `.dacpac` and maps SQL Server types to their Snowflake equivalents.

| SQL Server | Snowflake |
|---|---|
| `NVARCHAR(n)` | `VARCHAR(n)` |
| `DATETIME2` | `TIMESTAMP_NTZ` |
| `UNIQUEIDENTIFIER` | `STRING` (or `BINARY(16)`) |
| `IDENTITY(seed,inc)` | `IDENTITY(seed,inc)` |
| T-SQL stored procedures | `// TODO(...)` placeholders for manual rewrite |

```bash
sdt import --from sqlpackage-dacpac --source-path ./MyDb.dacpac --output ./my-project
```

> [!WARNING]
> This is best-effort. T-SQL stored procedures are emitted as placeholders — they must be rewritten by hand into Snowflake SQL or JavaScript. Manual review is required for any non-trivial schema.

---

## Terraform state

Reads a `terraform.tfstate` produced by the `Snowflake-Labs/snowflake` provider. Each `snowflake_*` resource maps to a matching SDT model object.

```bash
sdt import --from terraform-state --source-path ./terraform.tfstate --output ./my-project
```

**After import:** Terraform-managed resources that aren't pure schema (warehouses sized by policy, grants managed elsewhere) may need manual reconciliation. Compare against your live account to confirm coverage.

---

## Live account

The fastest start: reverse-engineer an existing Snowflake account directly into a project. This is `sdt extract` wrapped in a project-skeleton generator.

```bash
sdt import --from snowflake-account --source-path PROD --output ./extracted --connection prod
```

The result is the standard SDT folder structure populated from your live account. See [Extract](extract.md) for the underlying extraction options.

---

## Flat SQL files

Already keep your schema as a directory of `.sql` files? The importer walks the directory, shallow-parses each file to detect object kind and identifier, and buckets each into the SDT layout.

```bash
sdt import --from sql-files --source-path ./sql --output ./my-project
```

> [!NOTE]
> Files containing multiple objects or DML emit warnings during import. Split multi-object files or move DML into post-deploy scripts after importing.

---

## dbt

A dedicated migration command reads a dbt project and reports how it maps into SDT.

```bash
# Read dbt_project.yml + target/manifest.json + sources.yml, emit a migration report.
sdt migrate from-dbt ./my-dbt-project --output report.json

# Normalize identifier case and fill in unpinned database/schema names.
sdt migrate from-dbt ./my-dbt-project --identifier-case upper --default-database ANALYTICS
```

| Flag | What it does |
|---|---|
| `--manifest <path>` | Override the `target/manifest.json` path |
| `--identifier-case` | `preserve` (default) / `upper` / `lower` |
| `--default-database` / `--default-schema` | Fill in entries the manifest left unpinned |

You can also import a compiled dbt manifest as schema:

```bash
sdt schema import-dbt ./manifest.json --out ./my-project
```

> [!IMPORTANT]
> dbt models are templated SQL. The migration reports what maps cleanly and what needs manual attention — Jinja-heavy models, incremental logic, and macros generally need a human pass.

---

## Run both in parallel

You don't have to cut over in a single step. The recommended transition:

1. **Import into a fresh project** — the importer never touches your source tool, so both can coexist.
2. **Keep deploying with your existing tool** while you validate the SDT project: `sdt build` then `sdt compare ./my-project.sdtproj 'snowflake://prod'` should report no unexpected diffs.
3. **Switch reads first** — use SDT for `compare`, `drift`, `lint`, and `review` against production before you let it write anything.
4. **Cut over writes** once the diff is clean and your team is comfortable, and retire the old tool.

> [!NOTE]
> Snowflake also ships **DCM Projects**, its own native declarative-DDL change-management option. SDT exports to it (`sdt export dcm`) so you can author, review, and lint in SDT and deploy through DCM if you prefer.

---

**Next:** [FAQ](faq.md) · **Up:** [Documentation home](README.md)
