# 🔌 Connecting to Snowflake

> Set up a connection profile so SDT can extract, compare, and deploy against a live Snowflake account.

**On this page:** [The profiles file](#the-profiles-file) · [Pick an auth method](#pick-an-auth-method) · [Key-pair](#key-pair-recommended) · [External browser (SSO)](#external-browser-sso) · [Password](#password-dev-only) · [MFA](#mfa-password--token) · [OAuth](#oauth) · [Env-var placeholders](#env-var-placeholders) · [VS Code](#adding-and-testing-from-vs-code) · [Troubleshooting](#troubleshooting)

---

## The profiles file

A connection profile carries your account locator, auth credentials, and default role / warehouse / database / schema so day-to-day commands don't repeat them. SDT talks to Snowflake over the official driver, which it bundles — you don't install anything separately.

Profiles live at:

- `~/.sdt/profiles.json` (Linux/macOS)
- `%USERPROFILE%\.sdt\profiles.json` (Windows)

The file stores account, user, role, warehouse, database, schema, and auth-method metadata — **never** the raw secret. A key-pair profile looks like:

```json
{
  "version": 1,
  "profiles": [
    {
      "name": "dev",
      "account": "your-account.region",
      "auth": {
        "method": "KEY_PAIR",
        "username": "DEV_SERVICE_USER",
        "privateKeyPath": "/home/me/.sdt/sdt_key.p8",
        "privateKeyPassphrase": "env:SNOWFLAKE_PRIVATE_KEY_PASSPHRASE"
      },
      "role": "DEV_ROLE",
      "warehouse": "DEV_WH"
    }
  ]
}
```

> [!TIP]
> You rarely write this file by hand — `sdt connection add` (below) and the **SDT: Add Snowflake Connection** command in VS Code both write it for you.

> [!IMPORTANT]
> Never commit credentials. Keep private key files, passphrases, and tokens **outside the repo** and reference them with `env:VAR_NAME` placeholders (see [Env-var placeholders](#env-var-placeholders)). The profiles file holds metadata only; secrets resolve from env vars at runtime.

### What you'll need

- A **Snowflake account locator**, e.g. `xy12345.us-east-1.aws` or `myorg-myaccount`. The full form is `<account>[.<region>][.<cloud>]`.
- A **user** with access to the warehouse and target schema.
- A **role** with enough privileges. For read-only extract: `USAGE` on the warehouse, `USAGE` on the database/schema, and `SELECT` on objects. For deploy: also `CREATE` / `MODIFY` on the target schema.
- A **virtual warehouse** for query execution.
- An **authentication method** (below).

---

## Pick an auth method

| Method | When to use | Notes |
|---|---|---|
| **Key-pair (RSA)** | Automation, CI/CD, any production use | Strongly recommended. Stays exempt as Snowflake moves to mandatory MFA. |
| **External browser (SSO)** | Interactive dev with corporate SSO (Okta, Azure AD, …) | No secret on disk; pops a browser per session. |
| **Password** | Throwaway dev accounts only | Most tenants enforce MFA on password logins. Avoid for automation. |
| **MFA (password + token)** | Interactive dev on a non-SSO account | Per-call token entry; not usable from CI. |
| **OAuth** | An external provider already issues Snowflake bearer tokens | You fetch the token; SDT just uses it. |

For a brand-new setup, generate a key-pair and use it for both local dev and CI.

---

## Key-pair (recommended)

### 1. Generate an RSA key (one-time)

With OpenSSL (`winget install ShiningLight.OpenSSL`):

```sh
openssl genrsa -out $HOME/.sdt/sdt_key.pem 2048
openssl rsa -in $HOME/.sdt/sdt_key.pem -pubout -out $HOME/.sdt/sdt_key.pub
```

Optionally encrypt the private key (recommended). OpenSSL prompts for a passphrase — remember it:

```sh
openssl pkcs8 -topk8 -inform PEM -outform PEM \
  -in  $HOME/.sdt/sdt_key.pem \
  -out $HOME/.sdt/sdt_key.p8 \
  -v2 aes-256-cbc
```

### 2. Register the public key on the Snowflake user

Strip the `BEGIN`/`END` lines and join the body into one line, then run as a role that can `ALTER USER`:

```sql
ALTER USER MY_USER SET RSA_PUBLIC_KEY='<the joined public-key body>';
```

Verify — a non-NULL fingerprint means it took:

```sql
DESCRIBE USER MY_USER;
-- Look for RSA_PUBLIC_KEY_FP
```

### 3. Store the passphrase in an env var

```sh
export SNOWFLAKE_PRIVATE_KEY_PASSPHRASE="<the passphrase you set>"
```

### 4. Add the profile

```sh
sdt connection add \
  --name prod \
  --account xy12345.us-east-1.aws \
  --user MY_USER \
  --auth KEY_PAIR \
  --key $HOME/.sdt/sdt_key.p8 \
  --key-passphrase env:SNOWFLAKE_PRIVATE_KEY_PASSPHRASE \
  --role SDT_DEPLOY \
  --warehouse XS_OPS_WH \
  --database ANALYTICS \
  --schema PUBLIC
```

`--role` is the role SDT activates after login; it must be granted to the user.

### 5. Test it

```sh
sdt connection test prod
```

You'll see:

```
server version, current account, role, warehouse, database, schema
```

---

## External browser (SSO)

Best for interactive development on an SSO-enforced tenant. SDT opens a browser, you complete the IdP flow (Okta / Azure AD / Google / …), and Snowflake returns a session token. No PEM file required; tokens last as long as your IdP session.

```sh
sdt connection add \
  --name dev \
  --account xy12345.us-east-1.aws \
  --user my.name@company.com \
  --auth EXTERNAL_BROWSER \
  --role SDT_DEV \
  --warehouse XS_DEV_WH \
  --database DEV_ANALYTICS
```

---

## Password (dev only)

```sh
export SNOWFLAKE_PASSWORD="<password>"
sdt connection add \
  --name local \
  --account xy12345 \
  --user TEST_USER \
  --auth PASSWORD \
  --password env:SNOWFLAKE_PASSWORD \
  --role TEST_ROLE \
  --warehouse XS_WH
```

> [!NOTE]
> Most Snowflake tenants enforce MFA on password logins. If yours does, this fails with `MFA token required` — use key-pair or external browser instead.

---

## MFA (password + token)

```sh
sdt connection add \
  --name mfa-dev \
  --account xy12345 \
  --user MY_USER \
  --auth MFA \
  --password env:SNOWFLAKE_PASSWORD \
  --role MY_ROLE \
  --warehouse XS_WH
# At test/run time SDT prompts for the one-time token from your Duo / Okta app.
```

Not usable from CI — there's no human to enter the token. For automation, use key-pair or OAuth.

---

## OAuth

Use when an external OAuth provider (Okta, Auth0, …) issues Snowflake session tokens and you want SDT to consume them.

```sh
export SNOWFLAKE_OAUTH_TOKEN="$(your-token-fetching-script)"
sdt connection add \
  --name oauth-prod \
  --account xy12345 \
  --user MY_USER \
  --auth OAUTH \
  --password env:SNOWFLAKE_OAUTH_TOKEN \
  --role SDT_DEPLOY \
  --warehouse XS_OPS_WH
```

> [!NOTE]
> The OAUTH bearer token is supplied through `--password` (there is no separate
> `--token` flag on `sdt connection add`). SDT does not refresh the token for
> you — fetch a fresh one before each run.

---

## Env-var placeholders

Any secret-bearing flag accepts an `env:VAR_NAME` placeholder. SDT reads the value from the named environment variable **at runtime**, not when the profile is written, so the secret never lands in `profiles.json`:

```sh
--key-passphrase env:SNOWFLAKE_PRIVATE_KEY_PASSPHRASE
--password env:SNOWFLAKE_PASSWORD
--password env:SNOWFLAKE_OAUTH_TOKEN   # OAUTH supplies its token via --password
```

On Windows, secrets can also resolve from the Credential Manager via the `keyring:<name>` placeholder.

---

## Adding and testing from VS Code

You don't have to use the CLI. From the Command Palette (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>):

- **SDT: Add Snowflake Connection** — walks you through creating a profile.
- **SDT: Test Snowflake Connection** — verifies an existing profile against the account.

The equivalent CLI commands:

```sh
sdt connection add --name <name> [flags]   # create a profile
sdt connection test <name>          # verify it
sdt connection list                 # list profiles (secrets redacted)
```

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `390101` | Public key on the user doesn't match the private key SDT is using. Re-check the `BEGIN`/`END` strip step. |
| `390100` | Role not granted to the user, or the role doesn't exist. |
| `390201` | User account locked from too many failed attempts. Unlock via SecurityAdmin. |
| `MFA token required` on password auth | Tenant enforces MFA. Switch to key-pair or external browser. |
| `OCSP_RESPONSE_TIMEOUT` | A proxy is blocking the OCSP check. Enable client-session keep-alive and/or configure the OCSP cache. |
| `Statement timeout / Warehouse is suspended` | Resume the warehouse (`ALTER WAREHOUSE <wh> RESUME;`) or set `AUTO_RESUME = TRUE`. |

---

**Next:** [Projects & suites](projects.md) · **Up:** [Documentation home](README.md)
