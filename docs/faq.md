# ❓ FAQ

> Straight answers to the questions people ask before and while adopting SDT.

**On this page:** [About the product](#about-the-product) · [Beta & pricing](#beta--pricing) · [Using it](#using-it) · [Data safety](#data-safety)

---

## About the product

### How is SDT different from schemachange, dbt, Flyway, or Terraform?

SDT is **state-based**: you author each Snowflake object as a `.sql` file describing the desired end state, and SDT computes the deltas. You never hand-write forward/backward migration scripts. Every alternative differs on one of those points:

| Tool | What it is | How SDT differs |
|---|---|---|
| **schemachange** | Snowflake migration runner — applies numbered scripts (`V001__…sql`) in order | Migration-based, not state-based. No compare engine, no drift detection, no safety classifier, no built-in refactor (rename/move) support. SDT is the declarative superset. |
| **dbt** | Transformation framework | dbt explicitly does **not** own DDL — your `CREATE TABLE` / `ALTER TABLE` changes happen outside dbt. SDT is the tool for that piece, and sits happily next to dbt. |
| **Flyway / Liquibase** | Migration-based change management | You write the forward and backward SQL by hand, and the adapters are generic SQL rather than Snowflake-native. SDT diffs desired state and classifies each change's safety per finding. |
| **Terraform** | Infrastructure-as-code | A general-purpose provider, not a schema compare/deploy engine. No 4-tier safety classification of DDL changes; no reviewable migration script. |

SDT does not replace dbt or your warehouse — it covers the schema compare/deploy/safety layer those tools leave out. For the schemachange migration path specifically, see [Migrating from other tools](migrating.md).

### Is SDT affiliated with Snowflake?

No. SDT is an **independent** tool. It is not affiliated with, endorsed by, or sponsored by Snowflake Inc. "Snowflake" is a trademark of Snowflake Inc., used here only to describe what SDT works with.

### Does my SQL leave my machine?

No. SDT runs entirely on your machine and talks directly to your Snowflake account. Your SQL, identifiers, table/column names, data, and credentials never leave your computer. Telemetry is opt-in and heavily sanitized, and AI features are bring-your-own-key — see [Data safety](#data-safety) and [PRIVACY.md](../PRIVACY.md).

---

## Beta & pricing

### What's free during the beta?

**Everything.** SDT is in a 30-day public beta, and during it every feature is free — the full deterministic engine plus all AI write-side features. See [Beta program](../README.md#beta-program) for the dates and details.

### What happens after the beta?

The deterministic core stays **free forever** — compare, safe deploy, extract, lint, format, lineage, Object Explorer, schema diagram, and the MCP server. After the beta, AI write-side features and some ownership features move to a paid Pro tier; if you keep using a Pro feature without a license, the CLI tells you clearly rather than failing silently. Free-tier commands never regress. The current tier breakdown is in the [README pricing section](../README.md#beta-program); release-time specifics land in [RELEASES.md](../RELEASES.md).

### Do AI features cost extra?

The AI features themselves follow the tier model above, but there is **no AI markup from us**. AI is **bring-your-own-key**: your prompts go directly from your machine to the AI provider you configure (Anthropic, OpenAI, Azure OpenAI, or an OpenAI-compatible / self-hosted endpoint), under your own account. SDT never proxies, resells, or stores that traffic, so you pay your provider directly at their rates and nothing to us for the tokens.

---

## Using it

### Do I need both the VS Code extension and the CLI?

No — either one works on its own. They run the same engine and share the same project format (`.sdtproj` + one `.sql` file per object), so you can author in VS Code and deploy from the CLI in CI, or do everything from one of them. Install whichever fits your workflow, or both.

### Can I use SDT in CI?

Yes. The CLI is built for it — compare, dry-run publish, lint, and (with a license) apply all run headless. Telemetry is automatically disabled when `CI=true`. See [CI/CD](ci-cd.md) for wiring patterns.

### What permissions does SDT need on my Snowflake account?

It depends on what you do:

- **Read-only extract / compare** — `USAGE` on the warehouse, `USAGE` on the database and schema, and `SELECT` on the objects you read.
- **Deploy** — additionally `CREATE` / `MODIFY` on the target schema.

SDT acts as the role you configure in the connection profile. Treat that profile like a CI/CD secret: its blast radius is everything that role can see and change.

### Does SDT work with my Snowflake edition?

Yes — SDT connects to Snowflake Standard, Enterprise, and Business Critical editions. It supports password, key-pair, external-browser SSO, MFA, and OAuth authentication. See [Connecting to Snowflake](connections.md).

---

## Data safety

![Safe deploy demo](../assets/demo-deploy.gif)

### Can SDT drop my tables?

Only when you explicitly tell it to. Destructive operations are blocked by default — they come out of a deploy as `-- WARNING:` comments, not executed SQL. To run one, you opt in with the matching gate: `allowDropTable`, `allowDropColumn`, `allowNarrowingTypes`, `allowTableRebuild`, or `allowUnrecoverableDrop`. This is by design — "no silent destruction" is a core principle. See [Safe deploy](safe-deploy.md) and the [Safety classifier](safety-classifier.md).

### What does error reporting send?

If you keep automatic error reporting on (opt-out, asked on first run), a report contains the error class, a **sanitized** message and stack (quoted identifiers and string literals are replaced before the report ever leaves your machine), the error kind, a fingerprint, and your OS / version info. It includes an email address **only** if you explicitly provide one.

It **never** sends your SQL text, identifier names, credentials, environment variables, file contents, or project structure. The sanitizer runs on your machine, so unsanitized data never reaches the network. Turn it off any time with `sdt telemetry off`, `SDT_TELEMETRY=0`, or `DO_NOT_TRACK=1`. Full detail: [PRIVACY.md](../PRIVACY.md).

---

**Next:** [Troubleshooting](troubleshooting.md) · **Up:** [Documentation home](README.md)
