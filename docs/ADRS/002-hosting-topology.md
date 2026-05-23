# ADR-002 — Hosting Topology: Single Fargate Service + Whitenoise

- Status: Accepted
- Date: 2026-05-23
- Deciders: kodex
- Related: docs/PRD.md · docs/ADRS/001-stack.md

## Context

ADR-001 chose Django 6 + HTMX with plain WSGI — no Channels, no frontend
service. We now need to fix the deployment shape on the existing `alvs-dev` AWS
environment to avoid provisioning new shared infrastructure. The PRD has zero
user-uploaded media in v1 (participant avatars come from `googleusercontent.com`),
so a full S3 + CloudFront static stack would solve a problem we do not yet have.

## Decision

- **Services**: ONE Fargate service named `hoy-puedo-web`. No frontend/backend
  split, no background worker.
- **Task size**: `0.25 vCPU` / `0.5 GB` RAM (Fargate minimum). Adequate for
  projected v1 load.
- **Task count**: DEV `desiredCount=1`. PROD `desiredCount=2` (HA + zero-downtime
  rolling deploys). PROD remains `desiredCount=0` until go-live, per the SROA
  precedent (infra ADR-A-001).
- **Cluster**: existing `alvs-dev` (DEV) and `alvs-prod` (PROD when ready).
- **Container base**: `python:3.13-slim`. WSGI via `gunicorn` (NOT daphne — no
  Channels per ADR-001).
- **Static assets**: served by Whitenoise from inside the container.
  `collectstatic` runs at image build time. Compressed + far-future
  `Cache-Control` headers via Whitenoise's manifest storage. NO S3, NO
  CloudFront for v1.
- **Media**: none. No S3 bucket provisioned. Zero user uploads in v1; avatars
  come from Google's CDN via the OAuth profile URL.
- **Database**: new logical database `hoy_puedo` on the existing `alvs-dev-pg`
  RDS instance. No new RDS instance; inherits same encryption-at-rest and
  backup policy.
- **ALB**: new host-based listener rule on the existing `alvs-dev-alb`:
  `Host: hoy-puedo.dev.grupoalvs.com` → new target group `hoy-puedo-web-tg`
  (port 8000, health check `/health/`).
- **DNS**: new `A`-alias record in the existing Route53 hosted zone
  `grupoalvs.com`: `hoy-puedo.dev.grupoalvs.com` → `alvs-dev-alb`. Wildcard
  cert `*.dev.grupoalvs.com` already covers it.
- **Secrets Manager**: three secrets under `alvs/dev/hoy-puedo/`:
  - `alvs/dev/hoy-puedo/django` — `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`
  - `alvs/dev/hoy-puedo/db` — `DATABASE_URL` (points at `alvs-dev-pg` /
    `hoy_puedo`)
  - `alvs/dev/hoy-puedo/cognito` — `COGNITO_DOMAIN`, `COGNITO_CLIENT_ID`,
    `COGNITO_CLIENT_SECRET`, `COGNITO_USER_POOL_ID`
- **CloudWatch log group**: `/ecs/alvs/dev/hoy-puedo` — 30-day retention (DEV),
  90-day (PROD).
- **Health endpoint**: `/health/` returns `200 OK` with a one-line body. Used by
  the ALB target group. No DB query — stays green during transient DB issues. A
  separate `/ready/` can be added later if needed.

## Consequences

- Container holds the static bundle, making the image a few hundred KB larger
  than the S3 alternative. Trivial at this scale.
- No CloudFront means no edge cache and no cache-invalidation step in CI/CD —
  less pipeline complexity.
- Adding user-uploaded media later (e.g., custom event covers in v2) requires
  retrofitting S3: bucket, IAM, django-storages, and a migration. Acceptable
  trade for v1 simplicity.
- `hoy_puedo` shares `alvs-dev-pg` with `sroa`. Connection-budget headroom is
  fine for v1 but worth tracking once both apps serve live traffic.
- `0.25 vCPU / 0.5 GB` is the cheapest Fargate increment. If gunicorn workers +
  Django footprint push past ~400 MB resident, bump to `0.5 vCPU / 1 GB`.
- PROD with `desiredCount=2` survives single-task crashes and supports rolling
  deploys without downtime.

## Alternatives Considered

- **SROA-like S3 + CloudFront for static** — proven but overkill for ~50 KB of
  static and zero media uploads. Rejected: solving a problem we don't have.
- **S3 only, no CloudFront** — extra moving parts, no edge benefit. Rejected:
  worst of both options.
- **Web + background worker (django-q2 or Celery)** — PRD §11 forbids
  notifications in v1, so there are zero async jobs. Rejected: YAGNI.
- **Single-task PROD (`desiredCount=1`)** — saves ~$4/mo but rolling deploys
  would cause downtime. Rejected: no HA.

## Open Follow-ups

- ADR-003 will fix the auth flow: callback URL, allauth settings, Cognito
  client provisioning steps (including the `ClientSecret` blocker from infra
  ADR-A-009).
- ADR-004 will fix CI/CD: GitHub Actions OIDC trust policy, ECR repo name,
  migrations one-shot task.
- Infra prerequisite (out of band): create the ALB rule, target group, Route53
  record, Secrets Manager entries, ECR repo `alvs/hoy-puedo`, and CloudWatch
  log group. Track in a separate INFRA.md checklist at deploy time.
