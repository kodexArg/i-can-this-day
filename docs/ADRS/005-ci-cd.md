# ADR-005 — CI/CD: PR-gated tests, merge-to-main deploys DEV

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md · docs/ADRS/001-stack.md · docs/ADRS/002-hosting-topology.md · docs/ADRS/004-dev-login-bypass.md
- Mirrors / depends on: AWS infra docs cicd.md · ADR-A-005 (CloudTrail) · ADR-A-006 (one-shot migrations)

## Context

ADR-002 fixed a single Fargate service on `alvs-dev`. The AWS infra side already provides the deploy
plumbing: OIDC trust, `gha-deploy-dev` / `gha-deploy-prod` roles, and the one-shot migration pattern
(ADR-A-006). The remaining decisions for this app are (a) the pre-deploy gate strategy and (b) the
DEV vs PROD trigger model. We pick PR-gated tests as the merge prerequisite, merge-to-main as the
DEV deploy trigger, and `workflow_dispatch` with a required reviewer for PROD.

## Decision

### Repository policy

- `main` is a **protected branch**. Direct pushes are blocked. All changes land via PR.
- PR requirements: at least one approving review (self-approval acceptable for solo dev today;
  tighten when the team grows) + all required status checks green.
- Branch naming is informational, not enforced: `feat/`, `fix/`, `chore/`, `docs/` prefixes are
  conventional.

### PR-time workflow (`.github/workflows/ci.yml`)

- Triggers: `pull_request` (opened, synchronize, reopened) and `push` to PR branches.
- Steps in order — each must pass before the next:
  1. Checkout
  2. Set up Python 3.13 + `uv` cache
  3. Install deps via `uv pip install` (main + dev/test requirements)
  4. Lint: `ruff check` + `ruff format --check`
  5. Unit + integration tests: `pytest`
  6. Playwright smoke tests: end-to-end runs of the three PRD §9 user stories (Marcos, Patricia,
     Claudia). Uses the dev-auth bypass from ADR-004 to skip the Cognito round-trip.
- All of the above are required status checks on `main`.

### Deploy-to-DEV workflow (`.github/workflows/deploy-dev.yml`)

- Triggers: `push` to `main` only (i.e., merge events). Does NOT re-run tests — trusts the PR gate.
- Steps:
  1. Checkout
  2. OIDC assume `gha-deploy-dev` (trust: `repo:kodexArg/i-can-this-day:ref:refs/heads/main`)
  3. ECR login on `789650504128.dkr.ecr.us-east-1.amazonaws.com`
  4. Docker build → push to `alvs/hoy-puedo` with tags `dev-<sha>` and `latest-dev`
  5. `ecs run-task` one-shot on `alvs-dev` — `python manage.py migrate --noinput`; deploy gates on
     exit code (fail fast if migration fails), per ADR-A-006
  6. `ecs update-service --force-new-deployment` on `hoy-puedo-web` → rolling deploy
- No manual approval. No re-test.

### Deploy-to-PROD workflow (`.github/workflows/deploy-prod.yml`)

- Trigger: `workflow_dispatch` with input `promote_sha` (the DEV-validated SHA to promote).
- Steps:
  1. Job uses GitHub environment `prod` → required reviewer (kodex) + 5-minute wait timer.
  2. OIDC assume `gha-deploy-prod` (trust: `repo:kodexArg/i-can-this-day:environment:prod`).
  3. Re-tag only: `ecr batch-get-image` for `dev-<promote_sha>` → `ecr put-image` as
     `prod-<promote_sha>`. Same manifest digest as the DEV-validated image. No rebuild.
  4. Migrations on PROD are NOT run by the workflow. Admin runs the one-shot migrate task manually
     from the workstation after reviewing the SQL diff, per ADR-A-006.
  5. `ecs update-service` on `alvs-prod` cluster → rolling deploy.

### Image and naming conventions

- ECR repo: `alvs/hoy-puedo`.
- Tags: `dev-<sha>` and `latest-dev` for DEV; `prod-<sha>` for PROD. No `latest` tag on PROD —
  forces deterministic re-tag.
- CloudWatch log groups: `/ecs/alvs/dev/hoy-puedo` (30d retention),
  `/ecs/alvs/prod/hoy-puedo` (90d retention). Audit trail covered by ADR-A-005 (CloudTrail).

### Secrets and env in CI

- GitHub repo secrets needed: **none** — OIDC replaces long-lived AWS keys; no Cognito or DB
  secrets ever touch GitHub.
- GitHub repo variables (convenience): `AWS_ACCOUNT_ID=789650504128`, `AWS_REGION=us-east-1`,
  `ECR_REPO=alvs/hoy-puedo`.
- `DEV_AUTH_ENABLED` is set only in the `ci.yml` Playwright job environment, never in any deploy
  job and never in any task definition.

## Consequences

- Solo-dev overhead: every change needs a PR. Acceptable cost; `gh pr create` makes it cheap.
- Tests gate deploys via the merge requirement, not the deploy job itself — keeps DEV deploys fast
  (~2–3 min) by avoiding redundant re-runs.
- The dev-auth bypass from ADR-004 is now a load-bearing piece of the smoke test suite. If it
  breaks, smoke tests break and deploys halt. Accepted.
- PROD deploys are guaranteed to promote a SHA that already ran on DEV (same image digest,
  re-tagged). A "untested commit on PROD" is not possible by accident.
- No DEV deploy on PR branches — keeps DEV stable. Devs preview locally via `manage.py runserver`
  with the dev-auth bypass.
- Admin bypass of branch protection (force-push, admin override merge) degrades the safety to "trust
  the human". Avoid; document in README later.

## Alternatives considered

- **SROA-verbatim (push-to-main auto-deploys, no test gate)** — fastest iteration, but red tests can
  reach DEV. Rejected: silent quality regression.
- **SROA + test gate inside the deploy workflow** — gates DEV but re-runs tests redundantly per
  deploy. Rejected: PR gate is the cleaner place.
- **Auto-promote PROD via git tag (`v*.*.*`)** — clean release semantics but diverges from SROA's
  `workflow_dispatch` pattern with no real ergonomic gain. Rejected: org alignment.

## Open follow-ups

- INFRA.md (later) will hold the bootstrap checklist: create ECR repo `alvs/hoy-puedo`, register
  OIDC trust for `gha-deploy-dev`, create CloudWatch log groups, Secrets Manager entries, ALB target
  group + listener rule, Route53 record.
- When the team grows past one person, require reviews from a different account and remove
  self-approval.
- Add a deploy-failure notification path (Slack / email) — no notifications infra in v1.
- Consider a `deploy-dev-preview` workflow per PR (ephemeral environment) — out of scope for v1.
