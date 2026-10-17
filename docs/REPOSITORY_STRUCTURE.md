# QNSC Repository Structure

Which repository owns what, how code is laid out inside a product, and how shared code reaches the products.
For how the pieces behave at runtime see [PLATFORM_ARCHITECTURE.md](./PLATFORM_ARCHITECTURE.md); for the technology see
[TECH_STACK.md](./TECH_STACK.md).

## Principles

1. **One monorepo per product.** A product's API, worker, web app, database schema, infrastructure and runbooks live
   together, so one pull request can change a feature end to end and one `git log` explains it.
2. **Shared code is a versioned repository, never a copy.** Application packages, CI workflows and infrastructure
   modules each have one home, are released independently, and are consumed by version. A product that needs a fix in
   shared code changes the shared repository and bumps the version.
3. **Share the code, not the runtime.** Products depend on shared packages at build time. Each product keeps its own
   database, cache, services, secrets and deployment, so one product's outage or deploy cannot take another's down.
4. **Docs live next to what they describe.** Alert runbooks, architecture decisions and setup instructions are in the
   repository that owns the behaviour. This repository holds only what applies to the whole organisation.

## The repositories

All are in the `quynhonsemiconductor` organisation. Private repositories (internal tooling, design documents) are
not listed here.

### Products

| Repository | What it is |
| :--------- | :--------- |
| [`rova`](https://github.com/quynhonsemiconductor/rova) | Single-tenant project and portfolio management (work items, iterations, releases, test cases, reporting). NestJS API and worker, React SPA |
| [`opshub`](https://github.com/quynhonsemiconductor/opshub) | Internal IT and HR operations: assets, access requests, compliance, workforce, contracts, ISMS and QMS records. NestJS API and worker, React SPA |
| [`qnsc-kb-backend`](https://github.com/quynhonsemiconductor/qnsc-kb-backend) | Knowledge Base API and workers: FastAPI, Celery, PostgreSQL with pgvector for hybrid retrieval |
| [`qnsc-kb-frontend`](https://github.com/quynhonsemiconductor/qnsc-kb-frontend) | The Knowledge Base web app (React, Vite) |
| [`qnsc-landing`](https://github.com/quynhonsemiconductor/qnsc-landing) | The company marketing site (Astro, English and Vietnamese), on Cloudflare Pages |

### Shared platform

| Repository | What it is | How products consume it |
| :--------- | :--------- | :---------------------- |
| [`app-platform`](https://github.com/quynhonsemiconductor/app-platform) | Shared application packages: `identity`, `observability`, `platform-runtime`, `platform-http`, `platform-cache`, `platform-db`, `platform-jobs`, `platform-mail` | npm packages on GitHub Packages, one tag per package (`<package>-vX.Y.Z`); Renovate opens one grouped PR per product |
| [`ci`](https://github.com/quynhonsemiconductor/ci) | Reusable GitHub Actions workflows and composite actions: build, scan, attest, deploy, release, the aggregate "required" gates | `uses: quynhonsemiconductor/ci/...@v1` |
| [`tf-modules`](https://github.com/quynhonsemiconductor/tf-modules) | Versioned OpenTofu modules: network, RDS, cache, ECS service, secrets, Cloudflare Pages, tunnel, edge rules, observability agents and alert rules | `?ref=<module>-vX.Y.Z` |
| [`infra`](https://github.com/quynhonsemiconductor/infra) | Platform-level infrastructure: human access to AWS, the security baseline, shared network, Cloudflare zone governance, the Grafana Cloud stack, shared object storage, remote state | Read through remote state; products do not edit it |
| [`infra-template`](https://github.com/quynhonsemiconductor/infra-template) | The starting skeleton for a new product's infrastructure repository | Cloned once |
| [`gitops`](https://github.com/quynhonsemiconductor/gitops) | Desired state for the Kubernetes estate: one Helm chart, one values file per product and environment, ArgoCD applications | CI bumps image tags here; see [Direction](./TECH_STACK.md#direction) |
| [`.github`](https://github.com/quynhonsemiconductor/.github) | This repository: the organisation profile, issue and PR templates, `CONTRIBUTING`, `SECURITY`, the Renovate preset, these docs | Inherited automatically |

### Other engineering projects

| Repository | What it is |
| :--------- | :--------- |
| [`chipgraph`](https://github.com/quynhonsemiconductor/chipgraph) | An AI agent system for chip design: a chip project as a build graph of specs, RTL and tests, with deterministic EDA checks and human approval gates. Pre-alpha |
| [`mcp-tools`](https://github.com/quynhonsemiconductor/mcp-tools) | A Model Context Protocol server giving AI assistants access to GitHub, Microsoft 365, Rova and a browser, with each person's own permissions |
| [`vlsi_deep_training`](https://github.com/quynhonsemiconductor/vlsi_deep_training) | QSOC, an RV32IMC microcontroller designed block by block as a deep-training project |

### Forks and archived

| Repository | Status |
| :--------- | :----- |
| [`code-review`](https://github.com/quynhonsemiconductor/code-review) | **Fork** of an open-source code-review tool, pinned by commit; the pull-request AI review check in `ci` runs it |
| [`VLSIT_RTL_Generator_AI_Model`](https://github.com/quynhonsemiconductor/VLSIT_RTL_Generator_AI_Model) | **Fork** of a third-party RTL generation project |
| `solodesk` | **Archived** (retired 2026-10-10); private, read-only history |

## Inside a product repository

`opshub` and `rova` share one layout (the Knowledge Base backend is a Python project with the same top-level ideas):

```
<product>/
├── apps/
│   ├── api/            NestJS HTTP API (one process)
│   ├── worker/         NestJS background process: relays, scheduled jobs (no HTTP)
│   └── web/            React + Vite SPA, and apps/web/functions: the Cloudflare Pages Function proxy
├── libs/               shared backend code: platform glue and one module per domain area
├── db/                 Drizzle schema, SQL migrations, seeds
├── infra/
│   ├── live/{_shared,develop,prod}/    the values for each environment
│   └── modules/stack/                  the one composition module both environments call
├── docs/runbooks/      operational runbooks; docs/runbooks/alerts/ one page per alert
├── test/               cross-cutting specs and end-to-end tests
├── .github/workflows/  thin callers of the reusable workflows in `ci`
├── Dockerfile          multi-target: api, worker, migrator
└── docker-compose.dev.yml    the local stack (Postgres, Valkey, LocalStack, an OTel collector)
```

- `infra/modules/stack` is why `develop` and `production` cannot drift structurally: both call it, and the files under
  `infra/live/<env>` hold only values.
- The backend is the repository root package; `apps/web` is the one pnpm workspace member.
- Every product README says how to run it from a clean clone. OpsHub's and Rova's **Local stack** sections need only
  `cp .env.example .env` plus a few commands, and their CI runs those same commands on every pull request (the
  `local quickstart` job), so the instructions cannot rot.

## How shared code is versioned

| Shared thing | Version scheme | Where a product pins it |
| :----------- | :------------- | :---------------------- |
| App packages | per-package semver, tag `<package>-vX.Y.Z`, released by release-please | `package.json` (caret ranges) |
| Terraform modules | per-module tag `<module>-vX.Y.Z` | the `?ref=` on each `source` |
| CI workflows and actions | a moving major tag `v1` | `uses: ...@v1` |
| Products | one semver per repository, tag `vX.Y.Z`; the tag is what deploys to production | release-please |

## Where infrastructure lives (and is going)

Platform-wide infrastructure is in `infra`. Each product still keeps its own `infra/` directory holding its cluster,
database, cache and deploy roles; that pattern exists three times and has drifted, and the plan is to move each product
onto one shared product-profile module in `infra`, one at a time. The rule to follow meanwhile is in `infra`'s
[`docs/repository-boundaries.md`](https://github.com/quynhonsemiconductor/infra/blob/main/docs/repository-boundaries.md):
read it before adding infrastructure anywhere.

## Adding a new product

1. Start from `infra-template` for the infrastructure repository structure, state wiring and deploy roles.
2. Use the shared packages for what they own (sign-in, config validation, HTTP errors, cache, telemetry) instead of
   writing them again; if one almost fits, change `app-platform`.
3. Add thin workflow callers for CI, deploy and release from `ci`; do not copy workflow logic.
4. Add a `renovate.json` extending the shared preset, so version bumps arrive as pull requests.
5. Give it a README section that runs the product from a clean clone, and a CI job that runs those same commands
   (OpsHub and Rova show the pattern).
