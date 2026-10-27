# Golden Path CI Pipeline

`.github/workflows/golden-path-ci.yml` is a reusable workflow (`on: workflow_call`) that encodes the checks every service must pass before it ships. A service team adopts it with a single `uses:` line, so security and quality standards are defined once and inherited everywhere.

## Jobs

| Job | Runs when | What it does | Why it exists |
|---|---|---|---|
| `lint` | always | ESLint on `packages/backend` and `packages/frontend` | Catches style and obvious bugs before review |
| `test` | always | Jest with coverage for the backend; writes a coverage summary to the job summary | The backend `coverageThreshold` (80%) fails the job if coverage drops |
| `security-scan` | `run_terraform_plan` | `checkov -d infra --hard-fail-on HIGH` | Blocks insecure IaC before it reaches AWS |
| `terraform-plan` | `run_terraform_plan` | Authenticates to AWS via OIDC, runs `terraform init` and `terraform plan` on `infra/stacks/dev`, publishes the plan in the job summary and as an artifact | Shows reviewers exactly what infrastructure will change |
| `docker-build` | pull requests, after `lint` and `test` | Builds backend and frontend images without pushing | Validates the Dockerfiles on every PR |
| `terraform-apply` | `run_terraform_apply` | Applies the saved plan to `infra/stacks/dev` using OIDC credentials | Deploys infrastructure only from `main` |
| `build-and-push` | `build_and_push`, after `terraform-apply` | Builds both images, pushes them to the module-managed ECR repos and forces a new ECS deployment | Ships the application once the infrastructure exists |

Required checks for branch protection: `lint`, `test`, `security-scan`, `terraform-plan`.

## Inputs and secrets

| Name | Type | Default | Purpose |
|---|---|---|---|
| `node_version` | string | `"20"` | Node.js version for `lint` and `test` |
| `terraform_version` | string | `"1.7.0"` | Terraform version for plan/apply |
| `run_terraform_plan` | boolean | `false` | Enables `security-scan` and `terraform-plan` |
| `run_terraform_apply` | boolean | `false` | Enables `terraform-apply` |
| `build_and_push` | boolean | `false` | Enables `build-and-push` |
| `aws_role_arn` (secret) | string | none | IAM role assumed through OIDC |

## How to adopt it

Create `.github/workflows/todo-service-ci.yml` (or the equivalent for your service):

```yaml
name: Todo Service CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write   # required here: the OIDC token is only granted if the caller requests it

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

Apply and image push only run on pushes to `main`, never on pull requests.

## Configuring the OIDC role

1. In the repository go to **Settings → Secrets and variables → Actions → New repository secret**.
2. Create `AWS_ROLE_ARN` with the role ARN provided by your platform team (`arn:aws:iam::<account-id>:role/<role-name>`).
3. The caller passes it as `aws_role_arn`; the reusable workflow reads `secrets.aws_role_arn` and hands it to `aws-actions/configure-aws-credentials@v4`. No long-lived AWS keys are stored.
4. The IAM role's trust policy must allow this repository to assume it through GitHub's OIDC provider.

## Notes

- AWS jobs use `us-east-1`, where the module creates its ECR repos and ECS cluster. The Terraform state bucket lives in `us-east-2`.
- Do not set `cache: 'npm'` on `actions/setup-node`; use version tags (not SHAs) for actions.
- Until Step 3 removes the mock provider credentials from `infra/stacks/dev/main.tf` and enables the S3 backend, `terraform-plan` and the AWS jobs are expected to fail.
