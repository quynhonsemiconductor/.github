# QNSC Platform — Tech Stack

What the QNSC engineering platform **runs today**, layer by layer, and (briefly, at the end) where it is heading.
Everything in the tables was read from the default branches of the repositories listed in
[REPOSITORY_STRUCTURE.md](./REPOSITORY_STRUCTURE.md) on 2026-10-18; where a version is stated it is the version those
repos pin, and the repo is the authority if the two ever disagree. For how the pieces fit together at runtime see
[PLATFORM_ARCHITECTURE.md](./PLATFORM_ARCHITECTURE.md); to get set up see [ONBOARDING.md](./ONBOARDING.md).

This document describes what exists. It does not carry "was" or "planned" rows in the main tables: anything that is
not running yet is in [Direction](#direction).

## Runs today

### Application runtime

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Backend language | TypeScript 6 on Node.js 24 | `.nvmrc` in every TypeScript repo says `24` |
| Backend framework | NestJS 11 on Fastify 5 | Rova and OpsHub: one NestJS monorepo each, an `api` process and a `worker` process |
| ORM / migrations | Drizzle ORM 0.45, SQL migrations run by a one-shot `migrator` task before each rollout | |
| Web | React 19, Vite 8, TanStack Router and Query, Tailwind CSS 4, a typed OpenAPI client | An SPA per product, served from Cloudflare Pages |
| Knowledge Base backend | Python 3.11+ (3.13 in the image), FastAPI, SQLAlchemy + Alembic, Celery workers and beat, pgvector for hybrid retrieval | `qnsc-kb-backend`; Poetry for dependencies. It does not use the shared TypeScript packages |
| Knowledge Base frontend | React, TypeScript, Vite, Tailwind CSS (npm, Node 20+) | `qnsc-kb-frontend`; also on Cloudflare Pages |
| Marketing site | Astro 5, Tailwind CSS 4, static output, English and Vietnamese | `qnsc-landing`, Cloudflare Pages |

### Data

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Relational | PostgreSQL on Amazon RDS, major 18 (target; see the version table below) | Local stacks and CI use the same major and minor as RDS, as container images. Knowledge Base uses pgvector on top of PostgreSQL |
| Cache / coordination | Valkey (Redis-compatible) on ElastiCache: `develop` shares one node between products; a product's production stack uses the module's serverless mode, and stays disabled until the product goes live | Used for sessions, rate limits, pub/sub wake-ups and the cross-pod lock behind scheduled jobs. The Knowledge Base's Celery broker and cache run Valkey too (the Redis protocol) |
| Object storage | S3 or Cloudflare R2 (S3-compatible), chosen per product by configuration | Uploads go browser → presigned URL → bucket. Local development uses LocalStack |
| Queues | Postgres tables (the transactional outbox) read by worker relays | See [Workers and outboxes](./PLATFORM_ARCHITECTURE.md#workers-and-outboxes) |

**Data-store versions.** One version per engine, the same in every place it runs. Renovate
bumps the container images in one group (`data stores (postgres, pgvector, valkey)`), minor and patch
only, and keeps Valkey on the 8.1 line. Terraform's `engine_version` is **not** in that group: it is bumped
by hand in the same change, and each repository's consistency test fails until its major matches the compose
pin. This is decided, not an oversight.

| Store | Local compose and CI | Test containers (app-platform) | AWS target | AWS today |
| :---- | :------------------- | :----------------------------- | :--------- | :-------- |
| PostgreSQL | `postgres:18.6` (Debian, not Alpine: glibc collation, as on RDS) for OpsHub and Rova | `postgres:18.6` (Debian) | major 18 (currently 18.6) | RDS 17.9 (OpsHub, Rova prod and develop); 16.13 (Knowledge Base, develop) |
| pgvector | `pgvector/pgvector:0.8.2-pg18` (Knowledge Base). **Exception:** this image is PostgreSQL 18.4, not 18.6 (no pgvector image pins 18.6); same major | n/a | pgvector 0.8.2 (RDS 18.4 and later ship it) | 0.8.1 on RDS 16 |
| Valkey | `valkey/valkey:8.1.10-alpine` | `valkey/valkey:8.1.10-alpine` | ElastiCache `8.1` (same engine as local and CI) | ElastiCache 7.2.6 (runtime-dev, rova prod) |

Valkey is 8.1 everywhere. ElastiCache's "8.2" is open-source Valkey 8.1 plus the valkey-search module, so it is not a
separate target, and no `valkey/valkey:8.2` image is published. Do not "upgrade" to 8.2.

### Identity

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Staff sign-in | Microsoft Entra ID, through `@quynhonsemiconductor/identity` | One shared package owns the sign-in flow, the session store and the guard, so products do not each reinvent it |
| OpsHub | `identity` 8: Better Auth, the `staff` preset (Entra, tenant-checked), server-side sessions behind a single opaque cookie | No token ever reaches the browser. A passwordless test login exists for local development and CI only, and the configuration refuses it in production |
| Rova | `identity` 7: a server-side (BFF) Entra flow with an opaque session cookie | |
| Knowledge Base | Its own accounts and password login, with Entra sign-in configurable | |
| Authorization | Stays in each product (roles and permissions are product data) | The package authenticates; it does not decide what a user may do |

**Entra app-registration standard** (decided 2026-10-18; **rollout in progress**, so a product's existing registrations may not match yet). Two registrations per product per environment, never one that does both jobs:

| Registration | Used by | Credentials and permissions |
| :----------- | :------ | :-------------------------- |
| Sign-in app | The API only | Delegated permissions only (OpenID sign-in, no application permissions). No Graph application permissions |
| `<Product> <Env> Service` | The worker only | A **certificate** credential and no client secret. Application permissions only for what the worker does. Sending mail is `Mail.Send` limited to the product's mailbox through **Exchange RBAC**; `Mail.Send` is never consented in Entra. For SharePoint or OneDrive, `Sites.Selected` rather than `Sites.*.All` |

On both registrations:

- **Assignment required**: only assigned users and groups can sign in.
- **No personal app owners.** An app owner can add credentials to the app, which is a privilege path to whatever the
  app can do. Apps are managed through admin roles instead, and continuity comes from **two admin accounts**: the
  owner's own, and a break-glass account protected by phishing-resistant MFA. One person leaving never orphans an app.
- **Redirect URIs:** loopback (`localhost`) redirect URIs are allowed **only** on Dev web apps and on public-client
  (desktop or CLI) apps. A production web app registers none.

### Shared application packages (`app-platform`)

Published to GitHub Packages as `@quynhonsemiconductor/*`, versioned and released independently with release-please.
Products depend on them at build time only: **share the code, not the runtime.**

| Package | Owns | Used by |
| :------ | :--- | :------ |
| `identity` | Authentication on Better Auth: staff (Entra), public and organisation SSO presets, secure defaults, the NestJS session guard, a conformance kit | OpsHub (8), Rova (7) |
| `platform-runtime` | `.env` loading, environment validation and typed config, process bootstrap, health endpoints, graceful shutdown, leader-elected scheduled jobs | OpsHub, Rova |
| `platform-http` | Error taxonomy and status mapping, the global exception filter, pagination, client-address resolution, the Pages-Function proxy helper | OpsHub, Rova |
| `platform-cache` | The Valkey/Redis client: key prefixing, fail-open behaviour | OpsHub, Rova |
| `observability` | OpenTelemetry bootstrap, the logger, request and job context, metric instruments | OpsHub, Rova |
| `platform-db` | The PostgreSQL connection layer: pool, TLS, readiness ping, advisory lock, transactions | OpsHub |
| `platform-jobs` | Durable jobs on Postgres (transactional enqueue, worker-only handlers, schedules) | not yet adopted by a product |
| `platform-mail` | Email transport: the `EmailSender` contract, a Microsoft Graph transport, a non-production SMTP transport, the idempotent `mail.send` handler | not yet adopted by a product |

There is also a private, never-published `testing` package (the testcontainers harness the others share).

### Infrastructure

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Infrastructure as code | OpenTofu 1.9, state in a shared remote bucket with a lock table | Versioned modules in `tf-modules`, pinned by git tag. CI refuses an apply that would delete or replace DNS or identity resources unless a human opts in |
| Compute | Amazon ECS on Fargate (Graviton/arm64); OpsHub and Rova build each image for amd64 and arm64 | An `api` service, a `worker` service and a one-shot `migrator` task per product and environment. Images live in ECR with immutable tags |
| Ingress | Cloudflare: DNS, WAF and rate limiting for the company zone, Cloudflare Tunnel to the API, Turnstile available at the edge | There is **no load balancer** in front of the products. A `cloudflared` sidecar in the API task dials out to Cloudflare |
| Web hosting | Cloudflare Pages, with a Pages Function that proxies `/v1/*` and `/api/auth/*` to the API | Static assets and the proxy ship from the same project |
| AWS foundation | One AWS account with a security baseline (CloudTrail, Config, GuardDuty, Access Analyzer) and one shared network per environment | Owned by the `infra` repo |
| Human access to AWS | An IAM user per person that holds **no permissions of its own** and exists to carry an MFA device; people assume short-lived roles (developer: read-only plus a managed session to reach the `develop` database; administrator), only with MFA, and every assumption is logged | Machines never use these: CI authenticates through GitHub OIDC. Federating the existing Entra tenant to IAM is the stated next step once the team outgrows per-person users |
| CI → cloud trust | GitHub OIDC; no cloud credentials are stored in GitHub | |
| Environments | `develop` and `production` | `develop` deploys from every merge to `main`; `production` from a release tag, behind an approval |

### Observability

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Telemetry | OpenTelemetry SDKs in the app push OTLP to a collector sidecar in each ECS task | Metrics and traces |
| Backend | One Grafana Cloud stack shared by every product | Dashboards, alert rules and contact points are provisioned centrally; each product adds its own alert rule group |
| Logs | A FireLens (Fluent Bit) sidecar ships application logs to Grafana | CloudWatch was tried for application logs and removed on purpose |
| Infrastructure alarms | CloudWatch alarms and an SNS topic for signals only AWS can see (ECS task health, RDS, queues) | Application symptoms (error rate, latency, pool contention, job failures) alert from Grafana instead |

### Email

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Transport | Microsoft 365 through the Graph API, sending from one shared mailbox per product, behind the `platform-mail` contract | **Migration in progress.** Products still send through Amazon SES today and are moving to `platform-mail`; once a product has moved, its SES configuration and bounce plumbing go away |
| Non-production | An SMTP transport, or a transport that only logs | Local development never sends real mail |

### Secrets

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| Storage | AWS Secrets Manager: **one bundle (a single JSON secret) per product and environment**, named `<product>/<env>/app`, one key per value, read key by key by the ECS task definition | |
| Ownership | OpenTofu owns the container, the declared keys and the wiring into task definitions and IAM, and **never the values**. The owner sets the values from the product's runbook, out of band | Values never appear in Terraform state, code, logs or chat |
| Rotation | Most values are issued by other systems (Entra, Cloudflare, GitHub), so rotation happens at the issuer and the new value is written to the bundle. A **signing key** that a verifier must keep accepting during the change rotates with a `-previous` companion key: the verifier gets both first, then the signer moves, then the old key is dropped | There is no rotation automation; the database master password is the exception (RDS-managed) |

### Delivery and quality

| Layer | What we run | Notes |
| :---- | :---------- | :---- |
| CI/CD | GitHub Actions. Shared logic lives in `ci` as reusable workflows and composite actions, consumed at a major version tag | Product workflows are thin callers |
| Required checks | One aggregate gate per workflow (`Backend CI required`, `Web CI required`) that fails if any job it depends on was skipped, cancelled or failed | Closes the hole where GitHub counts a skipped required check as passing |
| Releases | release-please: Conventional Commits drive a release PR, merging it tags the release | Every package and product has its own changelog |
| Dependencies | Renovate with a shared preset in this repository; `@quynhonsemiconductor/*` updates arrive as one grouped PR per product | |
| Security scanning | Gitleaks, osv-scanner, Semgrep, Trivy on the image, actionlint and zizmor on workflows, CodeQL, SLSA provenance on built images | Blocking on critical findings |
| Code review | Human review from a code owner is mandatory; an AI review check also runs on pull requests (a pinned fork of an open-source reviewer) | |
| Packages | GitHub Packages for the shared npm packages | Reading needs a token; see [ONBOARDING.md](./ONBOARDING.md#3-the-github-packages-token) |
| Unit and integration tests | Vitest (TypeScript), pytest (Knowledge Base), Playwright for browser end-to-end | Integration tests run against real Postgres and Valkey containers, not mocks |

## Direction

Not running for the products yet, stated at a high level so a new engineer is not surprised by the repositories that
already exist for it.

- **Hybrid estate: cloud plus an on-premises Kubernetes (k3s) cluster.** The `gitops` repository holds the desired
  state: one Helm chart for every service, one values file per product and environment, and ArgoCD applications that
  reconcile them. A product repository never touches a cluster; CI builds and attests the image, then bumps an image
  tag in `gitops`. Rova is first and runs on both paths side by side until the cutover is proven and reversible.
- **Secrets on the cluster:** the same bundles, with the same key names, move to Azure Key Vault through External
  Secrets on the on-premises cluster (the `identity` package documents this). The External Secrets store checked into
  `gitops` points at AWS Secrets Manager today.
- **Infrastructure consolidation.** Each product repository still carries its own `infra/` directory; the plan is for
  one product-profile module in `infra` to replace the three copies.
- **Shared packages not yet adopted:** `platform-jobs` (durable jobs) and `platform-mail` (Graph email) are built and
  released; products move onto them one at a time.
