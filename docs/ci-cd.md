# 🔁 CI/CD integration

> Run SDT in your pipeline so every pull request previews schema changes and every merge deploys them safely.

**On this page:** [The CI pattern](#the-ci-pattern) · [Exit codes for gating](#exit-codes-for-gating) · [GitHub Actions](#github-actions) · [GitLab CI](#gitlab-ci) · [Azure DevOps](#azure-devops) · [Secrets in CI](#secrets-in-ci) · [Scheduled drift gate](#scheduled-drift-gate) · [PR diff comments](#pr-diff-comments) · [Docker image](#docker-image)

---

## The CI pattern

SDT — Snowflake Data Tools fits the standard four-stage pipeline. The same `.sdtpac` artifact you build in CI is the artifact you deploy, so what you reviewed is exactly what ships.

| Stage | Command | Runs on |
|---|---|---|
| **Build** | `sdt build` — compile `.sdtproj` to `.sdtpac` | every push + PR |
| **Compare** | `sdt compare` — diff project against the live account or last-deployed pac | every PR |
| **Review gate** | `sdt publish --dry-run` — show the migration without running it | every PR |
| **Publish** | `sdt publish --apply --yes` — deploy on merge | merge to `main` |

The rule of thumb: **dry-run in the PR, apply on merge.** A PR shows the diff and fails the build if it carries unapproved destructive changes; the merge job applies the same plan.

> [!IMPORTANT]
> Always build the pac once and pass it forward as a pipeline artifact. Re-building between the review job and the deploy job risks deploying something the reviewer never saw.

---

## Exit codes for gating

Every `sdt` command returns a stable exit code. Gate your pipeline on these.

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | User error (bad args, missing file, syntax error) |
| 2 | System error (network, permission, internal failure) |
| 3 | Compare diff found (drift detected in CI, flag-controlled) |
| 4 | Destructive changes detected and not approved |
| 5 | Drift detected (`sdt drift` in CI mode) |
| 6 | Locked feature attempted without a Pro/Team/Enterprise license |

In any non-interactive shell — when `stdin` is not a TTY, or `CI=true` — SDT never prompts and never opens a browser. Pass `--yes` (or `--confirm`) explicitly, and use `--output json` for machine-readable results.

> [!TIP]
> Exit code 4 is your destructive-change tripwire. A PR job that runs `sdt publish --dry-run --fail-on-destructive` fails the build before anyone can merge an accidental `DROP TABLE`.

---

## GitHub Actions

Install the published CLI and drive it directly — this works on any runner today, with no extra Marketplace dependency.

### PR validation — build, dry-run, fail on destructive

```yaml
name: Validate SDT project on PR
on:
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g @sdt-tools/cli

      # Build the artifact once.
      - run: sdt build --project ./db/MyProject.sdtproj --out ./bin/MyProject.sdtpac

      # Register the connection from env-var secrets (see "Secrets in CI").
      - run: sdt connection add --name staging
               --account "$SDT_ACCOUNT" --user "$SDT_USER"
               --auth KEY_PAIR --key ./key.p8
        env:
          SDT_ACCOUNT: ${{ vars.SNOWFLAKE_ACCOUNT }}
          SDT_USER: ${{ vars.SNOWFLAKE_USER }}

      # Dry-run the deploy; exit code 4 fails the build on unapproved destructive change.
      - run: sdt publish --pac ./bin/MyProject.sdtpac --connection staging --dry-run
```

### Deploy on merge

```yaml
name: Deploy SDT project
on:
  push: { branches: [main] }

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g @sdt-tools/cli
      - run: sdt build --project ./db/MyProject.sdtproj --out ./bin/MyProject.sdtpac
      - run: sdt connection add --name prod
               --account "$SDT_ACCOUNT" --user "$SDT_USER" --auth KEY_PAIR --key ./key.p8
        env:
          SDT_ACCOUNT: ${{ vars.SNOWFLAKE_ACCOUNT }}
          SDT_USER: ${{ vars.SNOWFLAKE_USER }}
      # Real deploy. --manifest captures a revert point.
      - run: sdt publish --pac ./bin/MyProject.sdtpac --connection prod
               --apply --yes --manifest ./deploy-${{ github.sha }}.json
```

> [!NOTE]
> A dedicated `GVOrganization/sdt-action` GitHub Action is planned but not yet published. Until it ships, use the CLI-in-CI pattern above — it is the supported way to run SDT in GitHub Actions today.

---

## GitLab CI

Run from the published Docker image so no per-job install is needed.

```yaml
stages: [build, deploy-dev, deploy-prod]
image: sdtddttools/sdt:0.2

variables:
  SDT_ACCOUNT: $SNOWFLAKE_ACCOUNT
  SDT_USER: $SNOWFLAKE_USER

build:
  stage: build
  script:
    - sdt build --project ./db/MyProject.sdtproj --out ./bin/MyProject.sdtpac
  artifacts: { paths: [./bin/*.sdtpac] }

deploy_dev:
  stage: deploy-dev
  script:
    - sdt publish --pac ./bin/MyProject.sdtpac --connection dev --apply --yes
  environment: dev

deploy_prod:
  stage: deploy-prod
  when: manual           # manual approval gate before production
  script:
    - sdt publish --pac ./bin/MyProject.sdtpac --connection prod --apply --yes --fail-on-destructive
  environment: prod
```

The `build` job publishes the `.sdtpac` as a GitLab artifact; the deploy jobs consume the same file.

---

## Azure DevOps

A multi-stage pipeline with a production approval gate. Install the CLI in each stage and pass the pac forward as a pipeline artifact.

```yaml
trigger:
  branches: { include: [main] }

stages:
  - stage: Build
    jobs:
      - job: BuildPac
        steps:
          - task: NodeTool@0
            inputs: { versionSpec: '20.x' }
          - script: npm install -g @sdt-tools/cli
          - script: sdt build --project ./db/MyProject.sdtproj --out ./bin/MyProject.sdtpac
          - publish: ./bin
            artifact: pac

  - stage: DeployProd
    dependsOn: Build
    jobs:
      - deployment: DeployProd
        environment: prod    # an AzDO Environment with an approval check
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: pac
                - script: npm install -g @sdt-tools/cli
                - script: >
                    sdt publish --pac $(Pipeline.Workspace)/pac/MyProject.sdtpac
                    --connection prod --apply --yes --fail-on-destructive
                  env:
                    SDT_ACCOUNT: $(SNOWFLAKE_ACCOUNT)
                    SDT_USER: $(SNOWFLAKE_USER)
```

Attach an **Approval and check** to the `prod` Environment so a human signs off before the deploy step runs.

> [!TIP]
> `sdt lint --format sarif` emits SARIF 2.1.0 — Azure DevOps and GitHub code-scanning both ingest it, so risky-deploy findings appear inline on the PR.

---

## Secrets in CI

Never inline credentials. Connection profiles resolve `env:<VAR_NAME>` placeholders at runtime, so the secret stays in your CI secret store.

```bash
# Profile references the env var; the value never touches disk or the profile file.
sdt connection add --name prod --account "$SDT_ACCOUNT" --user "$SDT_USER" \
  --password env:SF_PASSWORD

# Key-pair auth — point at a key file the runner materialized from a secret.
sdt connection add --name prod --account "$SDT_ACCOUNT" --user "$SDT_USER" \
  --auth KEY_PAIR --key ./key.p8 --key-passphrase env:SF_KEY_PASSPHRASE
```

Both `--password` and `--key-passphrase` accept the `env:VAR_NAME` form. The profile store records the placeholder, not the secret.

> [!WARNING]
> Mark the key file as a short-lived workspace artifact and let the runner clean it up. Never commit `*.p8` / `*.pem` files — SDT's project `.gitignore` excludes them by default.

---

## Scheduled drift gate

Run `sdt drift-gate` on a schedule to catch a warehouse that has wandered from its declared state — including replica accounts drifting from a primary.

```yaml
name: Nightly drift gate
on:
  schedule:
    - cron: '0 6 * * *'   # 06:00 UTC daily

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g @sdt-tools/cli
      # Fail (exit 5) if any replica drifts from the primary by more than 0 differences.
      - run: sdt drift-gate --primary prod-us --replicas prod-eu,prod-apac --threshold 0
```

For a single account, `sdt drift --project ./MyProject.sdtproj --connection prod` reports drift against the project and exits 5 when drift is found.

---

## PR diff comments

Post the schema diff straight onto the pull request. `sdt pr-comment` renders the project diff, lint findings, and impact analysis as a GitHub-flavoured Markdown comment (Team tier).

```yaml
- run: npm install -g @sdt-tools/cli
- run: sdt pr-comment --base main --head HEAD --format md > comment.md
- uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: fs.readFileSync('comment.md', 'utf8'),
      });
```

`sdt install-hooks` installs a matching pre-commit hook that runs `sdt validate` + `sdt lint` on changed `.sql` files, so the same checks fire before the commit ever reaches CI.

---

## Docker image

The published image bundles the `sdt` CLI for any container-based pipeline.

| Tag | Registry |
|---|---|
| `sdtddttools/sdt:0.2`, `:0.2.0`, `:latest` | Docker Hub |
| `ghcr.io/sdtddttools/sdt:0.2` | GHCR |

Multi-arch (`linux/amd64`, `linux/arm64`). The image runs as a non-root user (`sdt`) with `tini` as the init process.

```bash
# Build a pac from a mounted project.
docker run --rm -v "$PWD":/work sdtddttools/sdt:0.2 \
  build --project /work/MyProject.sdtproj

# Deploy, passing credentials as env vars.
docker run --rm -v "$PWD":/work -e SDT_ACCOUNT=... -e SDT_USER=... \
  sdtddttools/sdt:0.2 \
  publish --pac /work/bin/MyProject.sdtpac --connection prod --apply --yes
```

A `docker-compose` watch loop is handy for local development:

```yaml
services:
  sdt-watch:
    image: sdtddttools/sdt:0.2
    command: ['drift', '--project=/work/MyProject.sdtproj', '--connection=dev', '--watch']
    volumes: ['./db:/work']
    environment:
      SDT_ACCOUNT: ${SNOWFLAKE_ACCOUNT}
      SDT_USER: ${SNOWFLAKE_USER}
      SDT_PRIVATE_KEY_PATH: /secrets/key.p8
    secrets: [sf_key]
secrets:
  sf_key: { file: ./keys/key.p8 }
```

---

**Next:** [Migrating from other tools](migrating.md) · **Up:** [Documentation home](README.md)
