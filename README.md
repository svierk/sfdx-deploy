# 🚀 SFDX Deploy

This repository implements a simple GitHub composite action for deploying Salesforce metadata to a target org. It supports source directories, manifest (`package.xml`) and metadata component selectors, configurable Apex test levels, validation-only (`--dry-run`) deployments, and optional **delta deployments** (only the components that changed between two git refs) via the [sfdx-git-delta](https://github.com/scolladon/sfdx-git-delta) plugin — which makes it equally suitable for pull request validation and for the actual deployment to higher environments.

## Usage

### Validate a deployment on pull requests (dry-run)

After installing the SF CLI and authorizing the relevant org, a check-only validation could look like this:

> Check out with `fetch-depth: 0` so metadata deletions can be detected (see [Deleting metadata](#deleting-metadata-destructive-changes)). Set `destructive-changes: false` if you don't need this — then a shallow checkout is fine.

```yaml
jobs:
  validation:
    name: Validation
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - name: Install SF CLI
        uses: svierk/sfdx-cli-setup@main

      - name: Salesforce Org Login
        uses: svierk/sfdx-login@main
        with:
          sfdx-url: ${{ secrets.SFDX_AUTH_URL }}
          alias: target-org

      - name: Validate Deployment
        uses: svierk/sfdx-deploy@main
        with:
          source-dir: force-app
          target-org: target-org
          test-level: RunLocalTests
          dry-run: true
```

### Deploy metadata to an org

```yaml
      - name: Checkout
        uses: actions/checkout@v7
        with:
          fetch-depth: 0            # required so metadata deletions can be detected

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@main
        with:
          source-dir: force-app
          target-org: target-org
          test-level: RunLocalTests
```

### Delta deployment (only changed components)

Delta mode uses [sfdx-git-delta](https://github.com/scolladon/sfdx-git-delta) to deploy only the components that changed between two git refs. This requires the full git history, so check out with `fetch-depth: 0`. The generated delta manifest takes precedence over `source-dir`, `manifest` and `metadata`, and component deletions are applied as post-destructive changes (see [Deleting metadata](#deleting-metadata-destructive-changes)). If no deployable changes are detected, the step succeeds without deploying.

```yaml
      - name: Checkout
        uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - name: Deploy Changed Components
        uses: svierk/sfdx-deploy@main
        with:
          delta: true
          delta-from: origin/${{ github.base_ref }}
          delta-to: HEAD
          target-org: target-org
          test-level: RunLocalTests
```

### Deleting metadata (destructive changes)

A plain Salesforce source deploy is **additive** — it never deletes a component from the org just because its file was removed from the repo. This action improves on that: it detects metadata you deleted in the repository (via [sfdx-git-delta](https://github.com/scolladon/sfdx-git-delta)) and applies those deletions to the target org **after** the deployment, for both delta *and* normal full deployments. This behaviour is **on by default** (`destructive-changes: true`); set it to `false` for an additive-only deploy.

```yaml
      - name: Checkout
        uses: actions/checkout@v7
        with:
          fetch-depth: 0            # required so deletions can be detected

      - name: Deploy Metadata
        uses: svierk/sfdx-deploy@main
        with:
          source-dir: force-app
          target-org: target-org
          test-level: RunLocalTests
          # destructive-changes: true  # default — components deleted in the repo are deleted on the org
```

How it works:

- Deletions are derived from the git range `delta-from`..`delta-to`. In a full deployment `delta-from` defaults to the previous commit (`HEAD~1`) and `delta-to` to `HEAD`; set `delta-from` explicitly (e.g. a base branch or release tag) to compare against a different baseline.
- The full git history is required (`fetch-depth: 0`). On a shallow clone deletions cannot be detected and the action emits a warning instead of deleting.
- Everything runs as a **single, atomic deploy** (one test run): when deletions exist the action deploys via a generated `package.xml` plus the derived `destructiveChanges.xml`, since Salesforce requires post-destructive changes to be paired with a manifest.
- With `dry-run: true` the deletions are validated (check-only) but not applied.

## Inputs

| Name               | Required | Default | Description                                                                                   |
| ------------------ | -------- | ------- | --------------------------------------------------------------------------------------------- |
| `source-dir`       | no       |         | Comma-separated list of source directories to deploy.                                          |
| `manifest`         | no       |         | Path to a manifest (`package.xml`) file specifying the components to deploy.                   |
| `metadata`         | no       |         | Comma-separated list of metadata component names, e.g. `ApexClass,CustomObject:Account`.       |
| `destructive-changes` | no    | `true`  | Auto-detect metadata deleted in the repo (via sfdx-git-delta) and delete it on the org **after** the deploy. Works for full and delta deployments. Needs `fetch-depth: 0`. Set `false` for additive-only. |
| `delta`            | no       | `false` | Deploy only components changed between two git refs (via sfdx-git-delta). Needs `fetch-depth: 0`. |
| `delta-from`       | no       |         | Git ref to compare from when detecting changes/deletions, e.g. `origin/main`. Required in delta mode; defaults to `HEAD~1` for destructive changes on full deployments. |
| `delta-to`         | no       | `HEAD`  | Git ref to compare to when detecting changes/deletions.                                        |
| `target-org`       | no       |         | Username or alias of the target org. Not required if the default org is set.                   |
| `test-level`       | no       |         | `NoTestRun`, `RunSpecifiedTests`, `RunLocalTests` or `RunAllTestsInOrg`.                       |
| `tests`            | no       |         | Comma-separated Apex tests for the `RunSpecifiedTests` level.                                  |
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

The action prints a concise, human-readable table of the affected components (created, changed and deleted) to the log and folds a detailed, human-readable breakdown of the deployment result — including component errors, test failures and code coverage warnings — into a collapsible "Deployment details" group (or "Validation details" for dry-runs). When `step-summary` is enabled, the job summary additionally lists the deployed, modified and deleted metadata — including deletions applied via destructive changes — plus a failures table when the deployment did not succeed.

## References

The deployment option supported by this GitHub composite action can be found in the Salesforce CLI Command Reference here:

- [project deploy start](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm#cli_reference_project_deploy_start_unified)

## Releases

Latest release notes can be found on the [release page](https://github.com/svierk/sfdx-deploy/releases).

## License

The scripts and documentation in this project are released under the [MIT License](https://github.com/svierk/sfdx-deploy/blob/main/LICENSE).
