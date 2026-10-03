# Shared GitHub Actions + Infisical

Status: Done
Date: 2026-10-01

## Goal

1. One repo, `Xomware/github-actions` (public), holds every reusable workflow and composite
   action. App repos call it with short shim files pinned to `@v1`. Modeled on
   `Lumist-Labs/github-actions`.
2. Infisical Cloud is the only place secrets are stored and edited. Workflows load them at
   run time and push them where they're needed (Terraform `TF_VAR_*` → SSM, build-time config,
   iOS signing). GitHub Actions secrets go away entirely. Update a value in Infisical, re-run
   the deploy, done.

Lambdas are untouched: they keep reading SSM at runtime.

## Progress (2026-10-03)

- Done: every step. Every deploy, Terraform, TestFlight and utility workflow in both orgs reads Infisical.
- GitHub secrets: the 3 iOS repos and the Xomware org are cleared; only green-square's `GH_PAT` remains.
- App Store Connect: key ZDDUD287XG (Admin; cloud signing needs it) and issuer in `/shared`.
- iOS app config recovered from GitHub secrets into `/xomfit` and `/xomper` as `IOS_*`.
- xomper SNS APNs key: set by CLI with the team key (A5X4MKX38D); Terraform ignores it (provider bug).
- Follow-ups outside this plan:
  - Delete the two deactivated IAM keys on domjgiordano after 2026-10-09.
  - Move the Spotify token exchange server-side; the client secret still ships in xomify's JS.
  - Retire xomtracks-frontend.

## What exists today (survey, 2026-10-01)

- 47 repos, 139 workflow files, ~8 archetypes copy-pasted with drift:
  - **Frontend S3/CloudFront:** 6 variants.
  - **Python Lambda deploy:** 5 variants. xomify is the most hardened; xomper keeps extra layers.
  - **Terraform:** one shape with ~8 copies plus 3 drifted ones.
  - **add-to-board:** 27 byte-identical copies.
  - **claude-issues:** 16 near-identical copies.
  - **Other:** iOS TestFlight (3), update-all-layers (3), wait-for-terraform (2).
- AWS auth is already OIDC everywhere. There are no static keys.
- Secrets reach things through 3 routes:
  - **GitHub secrets:** roughly 55 distinct names. The main ones are `AWS_ROLE_ARN` (34 files), `BOARD_TOKEN`, `DEV_ANTHROPIC_API_KEY`, `NOTION_TOKEN`, the ASC keys, and the `TF_VAR_*` sources.
  - **`TF_VAR_*` → Terraform → SSM:** for example the xomify Spotify client secret and the xomper Supabase/Anthropic keys.
  - **SSM written by Terraform from computed values:** API IDs, Cognito pool/client IDs, URLs, and `random_password`s.
- Lambdas read SSM at runtime through a per-repo `lambdas/common/ssm_helpers.py`. It exists in xomify, xomper and today-in-sports; xomcloud uses `config.py`, and xomtracks/armchair call SSM inline.
- Frontends read SSM at build time and `sed` the values into `environment.ts` or `NEXT_PUBLIC_*`.

## Infisical layout

Project `code` (the other project, personal, is out of scope). Environment `prod`.

```
/shared             cross-app values: ASC_*, APPLE_TEAM_ID, GA4_MEASUREMENT_ID
/aws                AWS_ROLE_ARN, AWS_TERRAFORM_PLAN_ROLE_ARN, AWS_TERRAFORM_APPLY_ROLE_ARN
/<app>              that app's secrets, e.g. /xomify: CLIENT_ID, CLIENT_SECRET, API_ACCESS_TOKEN
```

One folder per app, not per repo: xomify-frontend, -backend, -infrastructure, -ios all read
`/xomify`. Terraform workflows export the folder as `TF_VAR_<lowercased name>`.

Auth: one Infisical machine identity with OIDC auth, trusting
`token.actions.githubusercontent.com`, subject `repo:Xomware/*`. No stored credentials. Its
identity ID and the project slug aren't secrets, so they're hardcoded as defaults in the
shared action — callers pass nothing, and it sidesteps org variables not reaching private
repos on GitHub Free.

## What stays where

- Terraform-computed values (API IDs, Cognito IDs, URLs, `random_password`) stay in SSM. They're
  AWS outputs, not values anyone edits, and the Lambdas and frontend builds already read them
  there. Infisical holds only what a human sets.
- Out-of-band SSM placeholders (`serper-key = "PLACEHOLDER"`, xomfit `jwt-secret`, the
  `/xomware/shared/google-oauth/*` seed script, `/clt-dynasty/api/*` hand copies) move into
  Infisical and get written to SSM by Terraform like every other secret.

## Decomposition

Each row is one PR unless noted. "Pilot" means one repo goes first and the rest copy it once it's proven.

| # | Piece | Size | Needs |
|---|---|---|---|
| 0 | **Manual (Dom):** GitHub OIDC machine identity on project `code` | — | — |
| 1 | Create `Xomware/github-actions`: `load-infisical-secrets` composite action, actionlint CI, README, tag `v1` | ~150 | 0 |
| 2 | `tools/seed-infisical.sh`: copy current secret SSM values into Infisical. GitHub secrets are write-only, so it lists the ones Dom has to re-enter by hand | ~100 | 0 |
| 4 | Reusable `deploy-frontend-s3.yml`: loads `/<app>` as env, runs the caller's build command, syncs to S3 with the cache-control tiers, invalidates CloudFront by alias. Each repo keeps one small `scripts/write-env` that maps env → `environment.ts` | ~180 | 1 |
| 4b | Pilot on xomware-frontend, then migrate the other frontends one PR each | ~30 each | 4 |
| 5 | Reusable `deploy-lambda-python.yml`: xomify's retry/verify, xomper's keep-extra-layers, optional wait-for-terraform | ~200 | 1 |
| 5b | Pilot on xomify-backend, then the other zip-based backends. xomcloud (ECR) and meals (Node) stay bespoke for now | ~30 each | 5 |
| 6 | Reusable `terraform.yml`: plan → PR comment → apply, with `/<app>` exported as `TF_VAR_*`, `fmt -check`, and a pinned TF version input | ~150 | 1 |
| 6b | Pilot on xomforms-infrastructure (no secrets), then xomify-infrastructure, then the rest | ~30 each | 6, 2 |
| 7 | Move the out-of-band SSM values into Infisical + Terraform, per affected app | ~20 each | 6b |
| 8 | Reusable `ios-testflight.yml`: ASC keys from `/shared`, the caller's config-render step | ~150 | 1 |
| 9 | Delete the GitHub secrets for each repo once that repo's migration has passed a real deploy. Destructive, so it's confirmed per repo | script | all |

**Order:**
1. Steps 1–2, proven end to end by the first frontend pilot (4b) reading `/aws`.
2. Step 6, Terraform, so `TF_VAR` secrets are live in Infisical.
3. Step 5, backends; step 7 alongside each app's terraform migration.
4. Step 4, frontends.
5. Step 8, iOS.
6. Step 9, cleanup.

Steps 4, 5, 8 are independent of each other once 1 lands.

## Bugs found during the survey (fold in, don't fix separately)

- xomcloud-frontend's SSM fetch doesn't check for empty values. It's the old `echo $(aws …)` form, and the same bug as xomify #296.
- xomforms-backend and xomtracks-backend swallow deploy errors with `|| echo`.
- clt-dynasty `/clt-dynasty/api/*` params were copied by hand from xomper, outside Terraform.
- xomper / xomcloud / xomware deploy on node 18 but run CI on node 20.
- No Terraform workflow saves the plan file, so apply re-plans. The reusable workflow should save the plan and apply it.

## Out of scope

- add-to-board and claude-issues: unused (BOARD_TOKEN, NOTION_TOKEN, DEV_ANTHROPIC_API_KEY are dead). Not migrated.
- Float (fastlane match), vest-*, xomfit-garmin, xomcloud-backend (ECR), meals-backend (Node). They migrate their secrets to Infisical with the action directly, with no shared workflow until a second caller exists.
