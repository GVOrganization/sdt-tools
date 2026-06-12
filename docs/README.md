# 📚 SDT Documentation

> Everything you need to use **SDT — Snowflake Data Tools**, from first install to CI/CD automation.

![Snowflake Data Tools](../assets/hero.png)

## Start here

| | Page | What you'll learn |
|---|---|---|
| 🚀 | **[Getting started](getting-started.md)** | Install → connect → first project → first compare → first deploy, in ~15 minutes |
| 🔌 | **[Connecting to Snowflake](connections.md)** | Every auth method (key-pair, SSO, OAuth, password, MFA), profiles, troubleshooting |
| 📁 | **[Projects & suites](projects.md)** | The `.sdtproj` format, folder layout, variables, deployment profiles, `.sdtpac` artifacts |

## The core workflow

| | Page | What you'll learn |
|---|---|---|
| 📥 | **[Extract a live account](extract.md)** | Reverse-engineer an entire account into git-ready `.sql` files |
| 🔍 | **[Schema compare](schema-compare.md)** | Diff project ↔ pac ↔ live account in any direction, read the results |
| 🛡️ | **[Safe deploy](safe-deploy.md)** | Generate migration scripts, opt-in gates, rollback clones, deploy manifests |
| 🚦 | **[The safety classifier](safety-classifier.md)** | What DESTRUCTIVE / UNRECOVERABLE / EXPENSIVE / WARNING mean and how to opt in |

## Reference

| | Page | What you'll learn |
|---|---|---|
| ⌨️ | **[CLI reference](cli-reference.md)** | Every `sdt` command, grouped by workflow, with flags and exit codes |
| 🧩 | **[VS Code extension reference](vscode-extension.md)** | Every command, view, setting, context menu, and keybinding |
| ⚙️ | **[Configuration reference](configuration.md)** | Every `.sdtproj` option and VS Code setting |

## Going further

| | Page | What you'll learn |
|---|---|---|
| ✨ | **[AI features](ai-features.md)** | Bring-your-own-key setup, sketch from prose, safer alternatives, Ask SDT |
| 🔁 | **[CI/CD integration](ci-cd.md)** | GitHub Actions, GitLab CI, Azure DevOps patterns; drift gates; PR comments |
| 🚚 | **[Migrating from other tools](migrating.md)** | Coming from schemachange, dbt, Flyway/Liquibase, SqlPackage, or raw SQL |
| ⚖️ | **[How SDT compares](comparisons/README.md)** | Honest side-by-sides: schemachange, SSDT/SqlPackage, DCM Projects, dbt |

## When something goes wrong

| | Page | What you'll learn |
|---|---|---|
| ❓ | **[FAQ](faq.md)** | How SDT compares to other tools, pricing, permissions, data safety |
| 🩹 | **[Troubleshooting](troubleshooting.md)** | Symptom → cause → fix, plus how to report a bug well |

---

## I want to…

| Goal | Go to |
|---|---|
| Get from zero to a deployed change | [Getting started](getting-started.md) |
| Put my existing Snowflake account under version control | [Extract](extract.md) |
| See what changed between my project and production | [Schema compare](schema-compare.md) |
| Understand why my deploy was blocked | [The safety classifier](safety-classifier.md) → [Safe deploy](safe-deploy.md) |
| Allow a column drop / type narrowing on purpose | [Safety classifier → opt-in gates](safety-classifier.md) |
| Roll back a deploy | [Safe deploy → rollback](safe-deploy.md) |
| Run SDT in a GitHub Actions pipeline | [CI/CD integration](ci-cd.md) |
| Fail CI when production drifts from git | [CI/CD → drift gate](ci-cd.md) |
| Set up AI features with my own API key | [AI features](ai-features.md) |
| Find a command I can't remember | [CLI reference](cli-reference.md) — or run `sdt find <keyword>` / press <kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd> in VS Code |
| Switch from schemachange / dbt / Flyway | [Migrating from other tools](migrating.md) |
| Compare SDT against the tool I use today | [Comparisons](comparisons/README.md) |
| Report a bug | [Troubleshooting → Still stuck?](troubleshooting.md) or [SUPPORT](../SUPPORT.md) |

---

## Install

```sh
# VS Code extension
code --install-extension sdt-ddt-tools.sdt-vscode

# CLI
npm install -g @sdt-tools/cli
sdt --version
```

Or install **Snowflake Data Tools** from the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=sdt-ddt-tools.sdt-vscode).

> [!NOTE]
> SDT is in a **30-day public beta** — every feature is free during the beta. [What happens after?](../README.md#beta-program)

---

**Sibling product:** working with Databricks too? [**DDT — Databricks Data Tools**](https://github.com/GVOrganization/ddt-tools) is the same design and CLI surface for Unity Catalog.
