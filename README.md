# Xomware shared GitHub Actions

Reusable workflows and composite actions for Xomware repos. Callers pin `@v1`.

Secrets live in Infisical (project `code`, environment `prod`), not in GitHub secrets.
Workflows load them at run time with GitHub OIDC. To change a secret, edit it in Infisical
and re-run the deploy.

## Infisical layout

One folder per app (`/xomify`, `/xomper`, `/armchair`, ...), shared by that app's frontend,
backend, infra and iOS repos. Each holds:

| Secret | Used by |
|---|---|
| `AWS_FRONTEND_ROLE_ARN`, `AWS_BACKEND_ROLE_ARN`, `AWS_DEPLOY_ROLE_ARN` | Deploy workflows, picked with `role-key` |
| `AWS_TERRAFORM_PLAN_ROLE_ARN`, `AWS_TERRAFORM_APPLY_ROLE_ARN` | `terraform.yml` |
| Everything else, named after the Terraform variable in caps | `terraform.yml` exports each as `TF_VAR_<lowercase>` |

Values Terraform computes (API IDs, Cognito IDs, URLs, generated passwords) stay in SSM.
Frontend deploys read them with the `ssm` input.

The OIDC machine identity trusts `repo:{Xomware,domgiordano}/*` in both GitHub subject
formats (name-based and the immutable ID-based one new repos get).

## Reusable workflows

| Workflow | What it does |
|---|---|
| [`terraform.yml`](.github/workflows/terraform.yml) | Plan, comment the plan on PRs, apply the saved plan on the default branch. |
| [`deploy-lambda-python.yml`](.github/workflows/deploy-lambda-python.yml) | Deploy changed Lambdas from `lambdas/<name>`, rebuild the shared layer when `common/` or `requirements.txt` changes, verify each deploy by code hash. |
| [`deploy-frontend-s3.yml`](.github/workflows/deploy-frontend-s3.yml) | Build with Infisical + SSM values in env, sync to S3 with cache tiers, invalidate CloudFront by alias. |

Each file's header comment and `inputs:` are the contract. A caller is a short shim:

```yaml
on:
  push:
    branches: [master]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    uses: Xomware/github-actions/.github/workflows/deploy-lambda-python.yml@v1
    with:
      app: xomify
```

## Composite actions

| Action | What it does |
|---|---|
| [`load-infisical-secrets`](actions/load-infisical-secrets/action.yml) | Export one Infisical folder as env vars, optionally also as `TF_VAR_*`. |

## Releasing

Lint with actionlint and shellcheck installed (CI runs both). Tag `vX.Y.Z`, then move the
major tag: `git tag -f v1 && git push -f origin v1`. A change that breaks callers' inputs or
behavior is a new major.
