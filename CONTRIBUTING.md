# Contributing to QNSC

Thanks for contributing to a QNSC repository. This guide covers the
conventions shared across the QNSC-VN organization. Individual repos may add
their own `CONTRIBUTING.md` for project-specific setup — this document is the
baseline that applies everywhere unless a repo says otherwise.

These are internal, proprietary repositories. Contributions come from QNSC
engineers and approved partners under NDA/contract — this guide assumes you
already have repo access.

## Before you start

- Check existing issues and open PRs to avoid duplicate work.
- For anything non-trivial (new service, breaking change, architecture shift),
  discuss with the repo's owners (see `CODEOWNERS`) before writing code.
- Never commit secrets, credentials, customer data, or export-controlled /
  regulated material. See [`SECURITY.md`](./SECURITY.md) if you find any
  already committed.

## Branching

- Branch off the repo's default branch.
- Use a short, descriptive branch name that reflects the change.
- Keep branches focused and short-lived — one logical change per branch/PR.

## Commit messages — Conventional Commits

All commits must follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<optional scope>): <subject>

<optional body>

<optional footer>
```

Allowed types:

`feat`, `fix`, `chore`, `docs`, `style`, `refactor`, `perf`, `security`,
`test`, `build`, `ci`, `deps`, `revert`

Rules:

- Subject line is lowercase, imperative mood ("add", not "added"/"adds").
  No Start Case, PascalCase, or UPPER CASE subjects.
- Keep the subject under ~72 characters; put detail in the body.
- Reference the related issue in the footer where applicable
  (`Closes #123`, `Relates to #456`).

## Pull requests

- Fill out the repo's PR template completely — summary, related issues, type
  of change, and how it was tested.
- Keep PRs reasonably small and focused; split unrelated changes into
  separate PRs.
- **Every PR requires at least one approving review from a code owner and a
  passing CI run before merge** — this applies org-wide, no exceptions for
  "small" changes.
- Don't merge your own PR, even if you technically have permission, unless
  the repo's owner has explicitly said otherwise for a specific case (e.g.
  urgent hotfix with no other reviewer available).

## Code style and tests

- Follow the linting/formatting conventions already configured in the repo.
- Add or update tests for the behavior you changed.
- Don't reduce existing test coverage without explicit sign-off from a code
  owner.

## Reporting security issues

Do not open a public issue or PR for a security vulnerability. Follow the
process in [`SECURITY.md`](./SECURITY.md) instead.

## Code of Conduct

All contributors are expected to follow our
[Code of Conduct](./CODE_OF_CONDUCT.md).
