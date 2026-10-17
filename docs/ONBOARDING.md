# Onboarding: your first day as a QNSC engineer

A path from "I have a laptop" to "I merged a change and watched it deploy". It tells you what to ask for, what to install,
how to run each product, how changes ship and which conventions the reviews hold you to. It links to each repository's own
instructions instead of repeating them, because those are tested by CI and this page is not.

Read [PLATFORM_ARCHITECTURE.md](./PLATFORM_ARCHITECTURE.md) when you want the big picture, and
[REPOSITORY_STRUCTURE.md](./REPOSITORY_STRUCTURE.md) to find where things live.

## 1. Access to request

Ask your team lead, who will route it to the platform owners (the code owners in this repository's `CODEOWNERS`). Ask by
role; you do not need all of these.

| You are | You need |
| :------ | :------- |
| Everyone | A **GitHub** account in the `quynhonsemiconductor` organisation and a **Microsoft 365** account on the company tenant. The products sign you in with Microsoft Entra ID, so the same account opens the apps you are about to run and test |
| Product engineer | Write access to the product repositories you will work in, and a **Grafana Cloud** account to read dashboards, logs and traces for `develop` and `production` |
| Platform / infrastructure engineer | In addition: **AWS** access (a personal IAM user that carries your MFA device and holds no permissions itself, plus the role your job needs: read-only with a managed session to the `develop` database, or administrator), a role in the **Cloudflare** account, edit rights in Grafana, and membership of the reviewer group that approves `production` deploys |
| Anyone who needs to reach a deployed `develop` app | A role in that product (roles come from Microsoft Entra app roles, which a tenant administrator assigns) |

Never share an account, and never ask a colleague to paste a token or a secret to you in chat. If you need access to a
secret, ask the owner to add you to the process that holds it.

## 2. Toolchain

Versions are the ones the repositories pin; if a repo disagrees with this page, the repo is right.

| Tool | Version | Why |
| :--- | :------ | :-- |
| Git and the GitHub CLI (`gh`) | current | You will use `gh auth token` for package access and `gh pr` for pull requests |
| Node.js | **24** (every TypeScript repository has an `.nvmrc`) | Backends, web apps, tooling |
| pnpm | **10 or 11**. `rova` and `app-platform` pin pnpm 11; pnpm fetches the pinned version for you | Package manager for the TypeScript repositories |
| Docker with Compose v2 | current | The local stacks: PostgreSQL, Valkey, LocalStack (S3), an OpenTelemetry collector |
| OpenTofu | **1.9.x** (CI pins 1.9.1; `infra-template` has an `.opentofu-version`) | Only if you touch infrastructure code |
| AWS CLI v2 | current | Only for platform work: assume your role with MFA |
| Python 3.11+ with Poetry | | Only for `qnsc-kb-backend` work outside Docker; its README shows the Docker-only route |
| Node.js 20+ and npm | | `qnsc-kb-frontend` uses npm |

## 3. The GitHub Packages token

The shared `@quynhonsemiconductor/*` packages are published to GitHub Packages, which needs a token with `read:packages`
even to install. **Do not create a personal access token for this**; reuse your GitHub CLI login, and keep the token out
of every file that is committed.

```bash
gh auth login                  # once; add the package scope if it is missing:  gh auth refresh -s read:packages

# Once per machine. pnpm 11 and later IGNORE a token written in a project's .npmrc (that file is committed, so an
# environment variable in it could leak to a registry the repository names) but expand one in your user-level file.
grep -qs 'npm.pkg.github.com/:_authToken' ~/.npmrc || echo '//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}' >> ~/.npmrc

export NODE_AUTH_TOKEN="$(gh auth token)"   # in every shell that runs pnpm install
```

The line in `~/.npmrc` is a placeholder, not a secret: the token itself only ever lives in the environment variable.
Symptom of a missing token: `pnpm install` fails with `ERR_PNPM_FETCH_401 ... No authorization header was set`.

## 4. Run a product locally

Each product's README has the exact, tested steps. They start from a clean clone and use only the files in the repository.

| Product | Start here | Sign in locally with |
| :------ | :--------- | :------------------- |
| OpsHub | the **Local stack** section of [its README](https://github.com/quynhonsemiconductor/opshub#local-stack) | the passwordless test login and a seeded email |
| Rova | the **Local stack** section of [its README](https://github.com/quynhonsemiconductor/rova#local-stack) | the dev-login form and a seeded email |
| Knowledge Base | [`qnsc-kb-backend`](https://github.com/quynhonsemiconductor/qnsc-kb-backend#readme), then [`qnsc-kb-frontend`](https://github.com/quynhonsemiconductor/qnsc-kb-frontend#readme) | the bootstrap administrator described in the backend README |

A few things hold for all of them:

- You copy exactly one file, `.env.example` to `.env`. The values are local placeholders; never put a real or deployed
  secret in it.
- OpsHub and Rova are offset from each other's default ports, so both stacks can run at once (each README has the table).
  Both single-page apps want port 5173; each README says how to start the second one elsewhere.
- In OpsHub and Rova no email leaves a laptop and uploads go to a local S3 emulator: the local stacks use only local
  containers, so nothing you do there reaches a deployed environment unless you paste real credentials into `.env`,
  which you must not.
- If a README step fails on a clean clone, that is a bug: open an issue or fix the README in the same pull request.

## 5. How a change ships

1. **Branch** from `main` (trunk-based: `main` is the only long-lived branch; merge small changes within a day or two).
2. **Open a pull request** against `main`. Fill in the template. The **title** must be a Conventional Commit with a
   lowercase subject, for example `fix(web): keep the filter when the list reloads`; it becomes the squash commit and
   feeds the changelog.
3. **Required checks** run. Each workflow has one aggregate gate (`Backend CI required`, `Web CI required`) that fails if
   any job in it failed, was skipped or was cancelled. Read the log line `dependency results` to see which. In OpsHub and
   Rova the jobs include security scans, the OpenAPI contract diff, the migration upgrade path, a Docker build and the
   `local quickstart` run.
4. **A code owner approves.** Every pull request needs at least one approving review from a code owner, and you do not
   merge your own.
5. **Squash merge.** One pull request becomes one commit, whose subject is the PR title.
6. **`develop` deploys by itself** from that merge: infrastructure first, then the image, the database migration and the
   rollout, then a readiness probe. Watch the *Deploy* workflow run on your commit; it is a few minutes.
7. **release-please** keeps a release pull request open (`chore: release ...`) with the next version and changelog.
   Merging it creates the `vX.Y.Z` tag.
8. **`production` deploys from the tag**, using the image already built on `main`, after a reviewer approves the
   `production` environment. Nothing reaches production without that approval.

`develop` is for you to look at; `production` is for everyone else. A change that is safe to merge but not safe to show
users ships behind a flag or unrouted. Database migrations run before traffic moves, so write them to work with the
*previous* release for the minutes both are live.

If your change touches a shared repository (`app-platform`, `ci`, `tf-modules`), the pull request there is the real change;
products pick it up later by version. See [REPOSITORY_STRUCTURE.md](./REPOSITORY_STRUCTURE.md#how-shared-code-is-versioned).

## 6. Conventions

- **Conventional Commits** for commits and PR titles. Types: `feat`, `fix`, `chore`, `docs`, `style`, `refactor`, `perf`,
  `security`, `test`, `build`, `ci`, `deps`, `revert`. Lowercase imperative subject, about 72 characters; detail goes in
  the body. The full rules are in [`CONTRIBUTING.md`](../CONTRIBUTING.md).
- **Small, focused pull requests**; one logical change each. Add or update tests for the behaviour you changed, and do not
  lower coverage without a code owner's agreement.
- **No secrets, ever**: not in code, tests, commit messages, logs, screenshots, issues or chat. Secrets are created empty
  by infrastructure code, set by their owner out of band and injected into the running service by reference. If you find
  one committed, follow [`SECURITY.md`](../SECURITY.md) instead of opening a public issue.
- **Describe, do not enumerate, in public places.** This organisation's repositories are public: leave account and tenant
  identifiers, internal hostnames and addresses, and secret names out of documentation, issues and pull-request text.
- **Write the runbook with the alert.** Every alert links to a page in the owning repository's `docs/runbooks/alerts/`;
  operational procedures sit beside it in `docs/runbooks/`. Architecture decisions are recorded as ADRs
  (`app-platform/docs/adr`) and design notes (`infra/docs`).
- **Configuration is validated at start-up.** A process refuses to boot on a missing or malformed variable and names it. A
  new variable goes in the schema, in `.env.example` and in the infrastructure definition, together.

## 7. Your first week

1. Get the access in section 1; run `gh auth status` and `pnpm -v`.
2. Run OpsHub locally (section 4), sign in as two different seeded users and notice what each can and cannot do.
3. Read the **Local stack** README and one runbook for an alert in the same repository.
4. Pick a small issue or a documentation fix, take it through all eight steps of section 5, and watch the `develop`
   deploy.
5. Open Grafana, find the product's dashboard, and find your request in the logs.
