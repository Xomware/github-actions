# Xomware shared GitHub Actions

Reusable workflows and composite actions for Xomware repos. Callers pin `@v1`.

Secrets live in Infisical (project `code`, environment `prod`), not in GitHub secrets.
Workflows load them at run time with GitHub OIDC. To change a secret, edit it in Infisical
and re-run the deploy.

## Infisical layout

| Folder | Holds |
|---|---|
| `/shared` | Values used by more than one app: App Store Connect keys, `GA4_MEASUREMENT_ID` |
| `/aws` | `AWS_ROLE_ARN`, `AWS_TERRAFORM_PLAN_ROLE_ARN`, `AWS_TERRAFORM_APPLY_ROLE_ARN` |
| `/<app>` | One app's secrets, shared by its frontend, backend, infra and iOS repos |

## Composite actions

| Action | What it does |
|---|---|
| [`load-infisical-secrets`](actions/load-infisical-secrets/action.yml) | Export one Infisical folder as env vars for the rest of the job. |

```yaml
permissions:
  contents: read
  id-token: write
steps:
  - uses: Xomware/github-actions/actions/load-infisical-secrets@v1
    with:
      path: /xomify
```

## Releasing

Tag `vX.Y.Z`, then move the major tag: `git tag -f v1 && git push -f origin v1`. A change
that breaks callers' inputs or behavior is a new major.

The plan for migrating every repo: [`docs/features/shared-actions-infisical/PLAN.md`](docs/features/shared-actions-infisical/PLAN.md).
