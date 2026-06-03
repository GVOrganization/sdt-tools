# 🩹 Troubleshooting

> Find the symptom you're hitting, see the likely cause, and apply the fix.

**On this page:** [Connection failures](#connection-failures) · [Extract issues](#extract-issues) · [Compare surprises](#compare-surprises) · [Deploy failures](#deploy-failures) · [Extension issues](#extension-issues) · [Still stuck?](#still-stuck)

---

## Connection failures

Run `sdt connection test <profile>` first — it reports the exact server error.

| Symptom | Cause | Fix |
|---|---|---|
| `390101 (08001)` | The public key registered on the user doesn't match the private key SDT is using. | Re-register the public key: re-check the BEGIN/END strip step when copying the key body, then `ALTER USER MY_USER SET RSA_PUBLIC_KEY='…'`. Verify `DESCRIBE USER` shows a non-NULL `RSA_PUBLIC_KEY_FP`. |
| `390100` | The role isn't granted to the user, or doesn't exist. | Grant the role to the user, or fix the `--role` in the profile. |
| `390201` | User account locked from too many failed attempts. | Unlock via a SecurityAdmin role. |
| `MFA token required` on password auth | The tenant enforces MFA on password logins. | Switch to key-pair or external-browser auth. |
| `OCSP_RESPONSE_TIMEOUT` | A proxy is blocking Snowflake's OCSP certificate check. | Set `--client-session-keep-alive=true` and/or configure the OCSP cache. |
| `Statement timeout` / `Warehouse is suspended` | The virtual warehouse is suspended. | `ALTER WAREHOUSE <wh> RESUME`, or set its `AUTO_RESUME = TRUE` so it self-starts. |

> [!IMPORTANT]
> Secrets must use the `env:VAR_NAME` placeholder in the profile — raw secrets are rejected. If a connection can't find its credential, confirm the named environment variable is set in the shell that runs `sdt`. Full setup: [Connecting to Snowflake](connections.md).

---

## Extract issues

| Symptom | Cause | Fix |
|---|---|---|
| Grants are missing from extracted files | `GET_DDL` never includes grants, and grant extraction is not part of the current beta. | Manage grants outside SDT for now; this is a known beta limitation. |
| Masking / row-access policy attachments are missing | `GET_DDL` does not include policy attachments. | The extractor recovers these separately; ensure your role can read `SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES` (or the `POLICY_REFERENCES(...)` table function). |
| Some object types don't appear | A small number of types (e.g. some Cortex objects) aren't exposed by `GET_DDL`. | These are out of the current model — there's nothing to extract for them. |
| A procedure round-trips differently than the source | `GET_DDL` re-quotes dollar-quoted procedure bodies, which can change formatting. | This is cosmetic; SDT falls back to `SHOW`/`DESC PROCEDURE` for body recovery. Treat such diffs as formatting (see below). |
| Re-extracting wiped a manual edit | An object's live DDL changed, so extract rewrote that file. | Re-running extract preserves manual edits **where the DDL hasn't changed**; commit your project to git before re-extracting so you can review and merge changes. |

---

## Compare surprises

![Schema Compare demo](../assets/demo-compare.gif)

These are the "but they look the same!" cases. Snowflake's own behavior is usually the cause.

| Symptom | Cause | Fix |
|---|---|---|
| An object shows as modified but looks identical | Formatting / whitespace differences, or re-quoted procedure bodies. | Enable `ignoreFormattingDifferences` so compare uses a canonical form. |
| Names differ only by case and still flag | Unquoted Snowflake identifiers are upper-cased at parse time; quoted ones preserve case. SDT stores names exactly as Snowflake does and compares case by default. | Use `--ignore-case` (or `compare.ignoreCase: true`) for unquoted-identifier comparisons. |
| Audit columns (e.g. `_LOADED_AT`) always show as differences | Those columns legitimately differ between environments. | Set `ignoreColumnPatterns` to skip them. |
| A retention setting flags as changed when "nothing changed" | `DATA_RETENTION_TIME_IN_DAYS` is inheritable; an inherited value is **not** equal to the same value set explicitly. | Set the value explicitly on the object in your project if you want it to match, or accept the diff as a real (inherited vs explicit) difference. |
| Comparing two environments whose database/schema names differ flags everything | The names genuinely differ. | Use logical-name mapping: `--map src=tgt` (add `--map-rewrite-strings` to also rewrite fully-qualified names inside object bodies). |
| A Dynamic Table `TARGET_LAG` flags as changed (`'1 minute'` vs `'60 seconds'`) | Lag strings that mean the same thing are written differently. | Current versions normalize the lag string before comparing; if you still see it, the two values are genuinely different. |

---

## Deploy failures

Most "failures" here are SDT doing its job — refusing to destroy data silently.

| Symptom | Cause | Fix |
|---|---|---|
| `publish --apply` refuses, leaving destructive ops as `-- WARNING:` comments | **By design.** Destructive and unrecoverable changes are gated. | Review the safety report, then opt in with the matching gate: `allowDropTable`, `allowDropColumn`, `allowNarrowingTypes`, `allowTableRebuild`, or `allowUnrecoverableDrop`. See the [Safety classifier](safety-classifier.md). |
| A column type change is blocked as narrowing | The new type could truncate existing data. | If the data fits, opt in with `allowNarrowingTypes`. SDT can run a data-fit pre-flight probe to check first. |
| Deploy aborts citing a dependency cycle | Streams depend on tables; tasks can chain via `AFTER`. A cycle in those predecessors is illegal. | Break the cycle in your object definitions; SDT orders deploys by dependency tier plus per-object predecessors. |
| A deploy failed partway through | A step errored mid-run. | SDT walks completed steps in reverse and rolls back to your starting state where every step was reversible. Read the output, fix the failing object, and re-run. |
| You want one-command rollback after a successful deploy | — | Publish with `--manifest <path>` to capture an audit trail, then `sdt revert --manifest <path>` to replay the reverse SQL. |

See [Safe deploy](safe-deploy.md) for dry-run, apply, and rollback in depth.

---

## Extension issues

| Symptom | Cause | Fix |
|---|---|---|
| The extension doesn't activate | VS Code version too old, or it needs a reload. | Requires VS Code 1.90+. Reload the window (**Developer: Reload Window** from the Command Palette). |
| Pro features (property compare, script preview) are greyed out | Not signed in / no license. | During the beta everything is free — make sure you're on the latest extension version. |
| A command isn't found in the Command Palette | The extension didn't finish activating. | Reload the window; confirm **SDT — Snowflake Data Tools** is installed and enabled in the Extensions panel. |
| You need diagnostic logs for a bug report | — | Open **View → Output** and select the SDT channel; include the relevant lines (with anything sensitive removed) in your report. |

---

## Still stuck?

Report it — bug reports directly shape the beta. There are three channels, all triaged from the same place:

1. **VS Code** — run **SDT: Report a Bug…** from the Command Palette (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>). Use this for wrong-output bugs that don't throw an error.
2. **CLI** — run `sdt feedback "<what happened>"` from any terminal. Add `sdt feedback --contact you@example.com` to be emailed when the bug is fixed.
3. **GitHub Issues** — open one at the [issue tracker](https://github.com/GVOrganization/sdt-tools/issues/new/choose) and pick the **Bug report** template. Search [existing issues](https://github.com/GVOrganization/sdt-tools/issues) first.

**What to include:**

- SDT version (`sdt --version`, or the extension version from the Extensions panel)
- Your OS and, for the CLI, your Node.js version
- What you did, what you expected, and what actually happened
- Any error output, with anything sensitive removed

What you type via `feedback` / **Report a Bug** is sanitized before sending. For private reports, security issues, or data-deletion requests, email **sdt.ddt.tools@gmail.com**. Full guidance: [SUPPORT.md](../SUPPORT.md).

---

**Up:** [Documentation home](README.md)
