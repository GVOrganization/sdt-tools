# SDT vs SSDT / SqlPackage — "Database Projects for Snowflake"

If you're coming from SQL Server, the shortest accurate description of SDT is: **SSDT's Database Projects + SqlPackage, rebuilt natively for Snowflake.** This page maps the concepts and is honest about what doesn't carry over.

## TL;DR

- Pick **SSDT / SqlPackage** if: your database is SQL Server. (SDT does not target SQL Server at all.)
- Pick **SDT** if: you're on Snowflake and want the workflow SSDT taught you — declarative project, schema compare, generated deploy script, publish profiles — with Snowflake-native object support.

## Concept map

| SSDT / SqlPackage concept | SDT equivalent |
|---|---|
| `.sqlproj` database project | `.sdtproj` |
| `.dacpac` build artifact | `.sdtpac` (documented ZIP: manifest + model + source) |
| Schema Compare (`.scmp`) | `sdt compare` + VS Code compare view |
| `SqlPackage /Action:Publish` | `sdt publish` |
| Publish profiles (`*.publish.xml`) | `deploymentProfiles.<env>` overlays |
| SQLCMD variables `$(var)` | project variables + per-env profiles |
| Refactor log (rename ≠ drop+create) | refactor log |
| Pre/post-deploy scripts | `preDeployScriptPatterns` / `postDeployScriptPatterns` |
| Composite projects / references | Project Slice + Suite |
| Import from live database | `sdt extract` |
| `BlockOnPossibleDataLoss` | Four-tier safety classifier with per-category gates |

## What SDT has that SSDT never did

- **A graded safety model.** SSDT gives you one blunt switch (`BlockOnPossibleDataLoss`). SDT classifies every change `SAFE` / `EXPENSIVE` / `DESTRUCTIVE` / `UNRECOVERABLE` with per-finding remediation, and unrecoverable changes (e.g. dropping a Stream — its cursor cannot be reconstructed) refuse by default under a *separate* gate from ordinary drops.
- **Snowflake-native objects.** Streams, Tasks, Dynamic Tables, Pipes, Stages, File Formats, masking/row-access policies — modeled, compared, and deployed, with their platform-specific failure modes encoded.
- **Zero-copy clone rollback.** Optional pre-deploy clone for `SWAP`-based instant rollback — no SQL Server equivalent exists.
- **Pure Node.** No .NET dependency; runs identically on macOS/Linux/Windows and in any CI image.

## What doesn't carry over (yet, or ever)

- **Maturity.** SSDT has ~15 years of production hardening; SDT is in public beta. The compare engine is most thorough on tables; long-tail object types diff via DDL equality.
- **Visual table designer** — not shipped yet.
- **Microsoft backing.** SDT is solo-developed. Mitigations: artifact is Apache-2.0, format is plain SQL + JSON in your repo, installed CLIs keep working regardless.

## Migrating an existing DACPAC

```sh
sdt import --from sqlpackage-dacpac --source-path ./MyDb.dacpac --output ./MyProject
```

The importer reads the SQL Server model and scaffolds an `.sdtproj`; T-SQL-specific constructs that have no Snowflake equivalent are surfaced as warnings for hand-translation — type mapping is the part that genuinely needs your judgment. See [Migrating from other tools](../migrating.md).

---

*Independent tool, not affiliated with Microsoft. "SSDT", "SqlPackage", and "SQL Server" are trademarks of Microsoft Corporation.*
