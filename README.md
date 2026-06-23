# core-cloud-workflow-helm-ecr-actions

This repository contains actions for a full Helm chart workflow targeting Amazon ECR

- `Helm setup` - install Helm and, optionally, the `helm-unittest` plugin
- `ECR login` - assume an IAM role via OIDC and log in to Amazon ECR
- `Helm lint` - run `helm lint` to catch structural issues in a chart
- `Helm validate` - render templates (`helm template`) and run unit tests (`helm unittest`)
- `Helm package` - resolve dependencies and package a chart into a `.tgz`
- `Helm publish` - optionally validate the release tag, optionally create the repository, and push the chart to ECR

There are also two workflows that combine several of these actions into self-contained pipelines:

- `validation` - runs the lint, validate and package actions, and uploads the packaged chart as an artifact
- `publish-helm-ecr` - logs in, packages and publishes the chart to ECR

## Pre-requisites

Before calling the workflows you need the following in place:

- A valid Helm chart (a `Chart.yaml` with `name` and `version`)
- The chart `version` field must use **bare semver** (`1.0.0`, not `v1.0.0`)
- An **ECR repository named after your chart's `name`**. The publish workflow pushes to `oci://<registry>/<chart-name>`, so the repository name must match the chart name exactly. You can either provision the repository ahead of time (by whatever means your team uses) and leave `create_repository: false`, or set `create_repository: true` to have the workflow create it on first push (see `create_repository` below).
- An IAM role your repository can assume via GitHub OIDC, with the ECR permissions listed under [IAM](#iam) below
- For the `publish-helm-ecr` workflow, the calling job **must** grant `id-token: write` permission so the OIDC token can be issued
- If you use `helm unittest` (the default), a `tests/` directory in your chart containing test suites

## Usage

Workflows can be used as:

```yml
jobs:
  validation:
    uses: Home-Office-Digital/core-cloud-workflow-helm-ecr-actions/.github/workflows/validation.yml@1.0.0
    with:
      chart_directory: "chart"
      helm_version: ""
      run_unittest: true

  publish:
    uses: Home-Office-Digital/core-cloud-workflow-helm-ecr-actions/.github/workflows/publish-helm-ecr.yml@1.0.0
    permissions:
      id-token: write
      contents: read
    with:
      aws_role_to_assume: my-team-helm-publish-role
      aws_region: "eu-west-2"
      chart_directory: "chart"
      helm_version: ""
      validate_version_tag: true
      create_repository: false
    secrets:
      ecr_account_id: ${{ secrets.AWS_ACCOUNT_ID }}
```

The individual actions can be used directly in the same manner if you want to compose your own job:

```yml
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v6
      - uses: Home-Office-Digital/core-cloud-workflow-helm-ecr-actions/actions/setup@1.0.0
        with:
          helm_version: ""
      - id: login
        uses: Home-Office-Digital/core-cloud-workflow-helm-ecr-actions/actions/login@1.0.0
        with:
          aws_region: "eu-west-2"
          role_to_assume: my-team-helm-publish-role
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
      - id: package
        uses: Home-Office-Digital/core-cloud-workflow-helm-ecr-actions/actions/package@1.0.0
        with:
          chart_directory: "chart"
      - uses: Home-Office-Digital/core-cloud-workflow-helm-ecr-actions/actions/publish@1.0.0
        with:
          registry: ${{ steps.login.outputs.registry }}
          package_path: ${{ steps.package.outputs.package_path }}
          chart_name: ${{ steps.package.outputs.chart_name }}
          chart_version: ${{ steps.package.outputs.chart_version }}
          chart_directory: "chart"
          aws_region: "eu-west-2"
          create_repository: "false"
          validate_version_tag: "true"
```

## Workflow inputs

### `validation.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `chart_directory` | No | `.` | Directory containing the Helm chart |
| `helm_version` | No | `""` | Helm version to install (e.g. `v3.16.3`). Empty uses the runner's preinstalled Helm |
| `run_unittest` | No | `true` | Run `helm unittest`. Set `false` for charts with no test suites |

### `publish-helm-ecr.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `aws_role_to_assume` | Yes | - | IAM role to assume. A full ARN, or a bare role name when used with the `ecr_account_id` secret |
| `aws_region` | No | `eu-west-2` | AWS region of the destination ECR registry |
| `chart_directory` | No | `.` | Directory containing the Helm chart |
| `helm_version` | No | `""` | Helm version to install. Empty uses the runner's preinstalled Helm |
| `validate_version_tag` | No | `true` | Assert the git tag matches the chart `version`. See [Versioning](#versioning) |
| `create_repository` | No | `false` | Create the ECR repository if missing. Requires `ecr:CreateRepository` on the role |

| Secret | Required | Description |
| --- | --- | --- |
| `ecr_account_id` | No | AWS account ID of the ECR registry. Required only when passing a bare role name rather than a full ARN |

## Versioning

When `validate_version_tag` is `true` (the default), the publish workflow asserts that the git tag the run was triggered from matches the chart `version` in `Chart.yaml`. Both must use bare semver and match exactly:

- Single-chart repository: tag `1.0.0` must equal `Chart.yaml` version `1.0.0`
- Multi-chart repository: tag `<chart-name>/1.0.0`, where the portion after the final `/` must equal the chart version

If your release flow is not tag-driven, set `validate_version_tag: false` and manage version consistency yourself.

## IAM

The role passed to the publish workflow must trust GitHub's OIDC provider, scoped to your repository, and grant the following ECR permissions.

`ecr:GetAuthorizationToken` must be granted on `Resource: "*"` (it does not support resource-level scoping). The remaining actions can be scoped to the target repository ARN:

- `ecr:BatchCheckLayerAvailability`
- `ecr:DescribeRepositories`
- `ecr:InitiateLayerUpload`
- `ecr:UploadLayerPart`
- `ecr:CompleteLayerUpload`
- `ecr:PutImage`

If you set `create_repository: true`, the role additionally needs `ecr:CreateRepository`. If your role does not grant it , provision the ECR repository ahead of time and leave `create_repository: false`.

The calling job must grant `id-token: write` so the OIDC token can be issued, otherwise the role assumption will fail at the credentials step.

## Notes

- The push target repository is named after the chart's `name` field. Ensure the chart name and the ECR repository name are identical.
- Pin the workflow to a released version tag (`@1.0.0`) or the moving major (`@1`) to receive patches without changing your `uses:` line.
- The `setup` action installs Helm via `azure/setup-helm`.