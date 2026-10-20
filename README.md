# 🚀 SFDX Deploy

This repository implements a simple GitHub composite action for deploying Salesforce metadata to a target org. It supports source directories, manifest (`package.xml`) and metadata component selectors, configurable Apex test levels, validation-only (`--dry-run`) deployments, and optional **delta deployments** (only the components that changed between two git refs) via the [sfdx-git-delta](https://github.com/scolladon/sfdx-git-delta) plugin - which makes it equally suitable for pull request validation and for the actual deployment to higher environments.

## Usage

### Validate a deployment on pull requests (dry-run)

After installing the SF CLI and authorizing the relevant org, a check-only validation could look like this:

> Check out with `fetch-depth: 0` so metadata deletions can be detected (see [Deleting metadata](#deleting-metadata-destructive-changes)). Set `destructive-changes: false` if you don't need this - then a shallow checkout is fine.

```yaml
# Validate pull requests with `pull_request`, never `pull_request_target` - the
# latter would run fork code with this repository's org credentials.
on:
  pull_request:
    branches: [main]

# Least-privilege token: this pipeline only reads the repository.
permissions:
  contents: read

jobs:
  validation:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false # don't leave the GITHUB_TOKEN in .git/config for later steps

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@v1.1.2

      - name: Salesforce Org Login
        uses: svierk/sfdx-login@v1.4.2
        with:
          sfdx-url: ${{ secrets.SFDX_AUTH_URL }}
          alias: target-org

      - name: Validate Deployment
        uses: svierk/sfdx-deploy@v1.3.0
        with:
          source-dir: force-app
          target-org: target-org
          test-level: RunLocalTests
          dry-run: true
```

### Deploy metadata to an org

```yaml
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0            # required so metadata deletions can be detected
          persist-credentials: false

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@v1.3.0
        with:
          source-dir: force-app
          target-org: target-org
          test-level: RunLocalTests
```

### List inputs (comma-separated or multi-line)

The list inputs - `source-dir`, `metadata` and `tests` - accept a comma-separated string as well as a multi-line block, and the two notations can be mixed freely within one step. A comma-separated string keeps short lists on a single line, a block stays readable once a list grows:

```yaml
      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@v1.3.0
        with:
          source-dir: 'force-app,my-other-app' # comma-separated
          tests: | # multi-line block
            AccountServiceTest
            OpportunityServiceTest
            LeadConversionTest
          target-org: target-org
          test-level: RunSpecifiedTests
```

Surrounding whitespace and blank lines are ignored, so `'force-app, my-other-app'` is read exactly like the tightly packed `'force-app,my-other-app'`. Whitespace inside a value is kept, e.g. for a layout like `Layout:Account-Account Layout`.

### Delta deployment (only changed components)

Delta mode uses [sfdx-git-delta](https://github.com/scolladon/sfdx-git-delta) to deploy only the components that changed between two git refs. This requires the full git history, so check out with `fetch-depth: 0`. The generated delta manifest takes precedence over `source-dir`, `manifest` and `metadata`, and component deletions are applied as post-destructive changes (see [Deleting metadata](#deleting-metadata-destructive-changes)). If no deployable changes are detected, the step succeeds without deploying.

```yaml
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Deploy Changed Components
        uses: svierk/sfdx-deploy@v1.3.0
        with:
          delta: true
          delta-from: origin/${{ github.base_ref }}
          delta-to: HEAD
          target-org: target-org
          test-level: RunLocalTests
```

### Deleting metadata (destructive changes)

A plain Salesforce source deploy is **additive** - it never deletes a component from the org just because its file was removed from the repo. This action improves on that: it detects metadata you deleted in the repository (via [sfdx-git-delta](https://github.com/scolladon/sfdx-git-delta)) and applies those deletions to the target org **after** the deployment, for both delta *and* normal full deployments. This behaviour is **on by default** (`destructive-changes: true`); set it to `false` for an additive-only deploy.

```yaml
      - name: Checkout
        uses: actions/checkout@v7.0.1
        with:
          fetch-depth: 0            # required so deletions can be detected
          persist-credentials: false

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@v1.3.0
        with:
          source-dir: force-app
          target-org: target-org
          test-level: RunLocalTests
          # destructive-changes: true  # default - components deleted in the repo are deleted on the org
```

How it works:

- Deletions are derived from the git range `delta-from`..`delta-to`. In a full deployment `delta-from` defaults to the previous commit (`HEAD~1`) and `delta-to` to `HEAD`; set `delta-from` explicitly (e.g. a base branch or release tag) to compare against a different baseline.
- The full git history is required (`fetch-depth: 0`). On a shallow clone deletions cannot be detected and the action emits a warning instead of deleting.
- Everything runs as a **single, atomic deploy** (one test run): when deletions exist the action deploys via a generated `package.xml` plus the derived `destructiveChanges.xml`, since Salesforce requires post-destructive changes to be paired with a manifest.
- With `dry-run: true` the deletions are validated (check-only) but not applied.

## Inputs

| Name               | Required | Default | Description                                                                                   |
| ------------------ | -------- | ------- | --------------------------------------------------------------------------------------------- |
| `source-dir`       | no       |         | Source directories to deploy, comma-separated or multi-line.                                   |
| `manifest`         | no       |         | Path to a manifest (`package.xml`) file specifying the components to deploy.                   |
| `metadata`         | no       |         | Metadata component names, comma-separated or multi-line, e.g. `ApexClass,CustomObject:Account`. |
| `destructive-changes` | no    | `true`  | Auto-detect metadata deleted in the repo (via sfdx-git-delta) and delete it on the org **after** the deploy. Works for full and delta deployments. Needs `fetch-depth: 0`. Set `false` for additive-only. |
| `delta`            | no       | `false` | Deploy only components changed between two git refs (via sfdx-git-delta). Needs `fetch-depth: 0`. |
| `delta-from`       | no       |         | Git ref to compare from when detecting changes/deletions, e.g. `origin/main`. Required in delta mode; defaults to `HEAD~1` for destructive changes on full deployments. |
| `delta-to`         | no       | `HEAD`  | Git ref to compare to when detecting changes/deletions.                                        |
| `target-org`       | no       |         | Username or alias of the target org. Not required if the default org is set.                   |
| `test-level`       | no       |         | `NoTestRun`, `RunSpecifiedTests`, `RunLocalTests` or `RunAllTestsInOrg`.                       |
| `tests`            | no       |         | Apex tests for the `RunSpecifiedTests` level, comma-separated or multi-line.                   |
| `dry-run`          | no       | `false` | Validate the deployment without saving components (check-only deployment).                     |
| `ignore-conflicts` | no       | `false` | Deploy local files even if they overwrite changes in the org.                                  |
| `ignore-warnings`  | no       | `false` | Allow the deployment to complete successfully despite warnings.                                |
| `wait`             | no       | `33`    | Number of minutes to wait for the deployment to complete.                                      |
| `api-version`      | no       |         | Override the api version used for api requests, for example `59.0`.                            |
| `step-summary`     | no       | `true`  | Write a result section to the GitHub Actions [job summary](https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary). Set to `false` to avoid collisions with a custom workflow summary. |

> When the `tests` input is provided, the CLI runs the specified tests and the `test-level` input is ignored.

> If none of `source-dir`, `manifest` or `metadata` is provided (and delta mode is off), the CLI deploys the package directories defined in your `sfdx-project.json` (typically `force-app`).

## Outputs

| Name        | Description                                          |
| ----------- | --------------------------------------------------- |
| `deploy-id` | ID of the deployment job.                            |
| `status`    | Final status of the deployment (e.g. `Succeeded`).   |

The step fails if the deployment fails, while still printing the deployment result and setting the outputs.

The action prints a concise, human-readable table of the affected components (created, changed and deleted) to the log and folds a detailed, human-readable breakdown of the deployment result - including component errors, test failures and code coverage warnings - into a collapsible "Deployment details" group (or "Validation details" for dry-runs). When `step-summary` is enabled, the job summary additionally lists the deployed, modified and deleted metadata - including deletions applied via destructive changes - plus a failures table when the deployment did not succeed.

## Security & versioning

Every `uses:` reference in the snippets above is **pinned to an exact release version**, e.g. `svierk/sfdx-deploy@v1.3.0`. Do the same in your own pipelines:

- **Never reference a mutable ref** such as `@main` or `@v1`. It runs whatever code sits on that branch/tag at run time - with access to your org credentials - so a compromised or rewritten ref would run unnoticed.
- **Good - pin to an exact release tag** (`@v1.3.0`). Readable, concrete, and bumped through reviewed pull requests.
- **Strictest - pin to a full-length commit SHA** (`@a1b2c3d…`) with the version as a trailing comment. A SHA can never be re-pointed by the publisher; the cost is readability. Worth it for actions from publishers you don't control.
- **Enable [Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot) for `github-actions`** so those pins are bumped for you instead of silently ageing. Note that Dependabot only scans `.github/workflows` and a root `action.yml` - copy-paste snippets in a README are not covered and have to be updated by hand.

This applies to **all** actions your workflows reference - this action as well as `actions/*` and any other third-party action. Latest versions at the time of writing: [`actions/checkout@v7.0.1`](https://github.com/actions/checkout/releases), [`svierk/sfdx-cli-setup@v1.1.2`](https://github.com/svierk/sfdx-cli-setup/releases), [`svierk/sfdx-login@v1.4.2`](https://github.com/svierk/sfdx-login/releases), [`svierk/sfdx-deploy@v1.3.0`](https://github.com/svierk/sfdx-deploy/releases).

Beyond pinning, a few rules are worth copying into the workflow that calls this action:

- **Least-privilege `GITHUB_TOKEN`** - declare a `permissions:` block granting only what the job needs. This action itself needs no token at all, so `contents: read` is enough for the deploy job.
- **`persist-credentials: false` on checkout** - the token is not written to `.git/config`, so later steps (SF CLI, third-party actions) cannot reuse it. The delta and destructive-change detection reads the local git history only and works fine without persisted credentials.
- **Secrets travel as secrets** - pass them straight into an action input (`sfdx-url: ${{ secrets.SFDX_AUTH_URL }}`) and reference them in your own shell steps as **environment variables** (`"$TARGET_ORG"`), never by interpolating `${{ ... }}` into the script itself - that would allow command injection and can leak values into the log. Every input of this action is handed to the shell through an `env:` mapping for exactly that reason.
- **Validate pull requests with `pull_request`, never `pull_request_target`** - the latter runs with the base repository's secrets, which would let a fork execute its own code against your org.
- **Gate production behind a [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)** - required reviewers and environment-scoped secrets keep a non-dry-run deploy from being triggered accidentally.

## References

The deployment option supported by this GitHub composite action can be found in the Salesforce CLI Command Reference here:

- [project deploy start](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm#cli_reference_project_deploy_start_unified)

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-deploy/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-deploy/blob/main/LICENSE).
