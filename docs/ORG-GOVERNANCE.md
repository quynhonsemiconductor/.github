# Organization governance

How access, teams and branch protection work across this organization. This
document describes the MODEL; it never names people, emails or internal hosts.

## Org roles

| Role | Who | Why |
|---|---|---|
| Owner (×2) | Two break-glass accounts only | Org-level settings, team creation, ruleset edits. Day-to-day work never uses them. |
| Member | Everyone on staff who pushes code | Base membership; no special rights beyond what teams grant. |
| Outside collaborator | Bots only | Repository-scoped access for automation; never for people. |

## Teams

Teams are groups of people with the same access. The default shape is **by
area** (platform, ai, rova, opshub, kb, vlsi, embedded); a product team exists
only when it has its own people or needs different access.

Every area has two teams, children of the `engineering` parent team:

| Team | Permission | Purpose |
|---|---|---|
| `<area>` | **Write** | The people doing the work; push access. |
| `<area>-leads` | **Maintain** | Review authority: CODEOWNERS for the area's repos, and maintainer of both teams (roster changes without an org owner). |

No team gets **Admin** on any repository. Nobody is added as a **direct
collaborator** — access always flows through a team, so offboarding is one
roster edit.

## Branch protection (rulesets)

Every repository carries the same `main`-protection ruleset:

- require **1 code-owner approval** (CODEOWNERS: `* @quynhonsemiconductor/<area>-leads`);
- **dismiss stale approvals** when new commits land;
- **require approval of the most recent push**;
- required status checks per repository (tests, lint, security scans);
- **no force-push, no deletion** of the default branch;
- **tag protection** for release tags;
- bypass is granted only to the `merge-bypass` team, and only in
  `pull_request` mode (it can skip the review gate when a change must ship
  immediately; it cannot bypass required checks on protected tags).

## New-repository checklist

1. Name: lowercase kebab-case, product or area first (`<product>` / `<area>-<thing>`).
2. Visibility: public by default; private only with a stated reason (private repos do not get rulesets on the Free plan — use branch protection instead).
3. Add `.github/CODEOWNERS` (copy the org template from `.github`, set `* @quynhonsemiconductor/<area>-leads`).
4. Apply the `main`-protection ruleset.
5. Settings: squash-merge only, delete head branches, wiki off, Actions token permission read-only.
6. Security: secret scanning + push protection on, CodeQL (or the repo's scanner) on.
7. `LICENSE`, `SECURITY.md`, `renovate.json` (extends the org preset), `.gitleaks.toml`.
8. Wire the shared CI reusables from `ci` (pinned by digest).
9. If it deploys: a `production` environment with required reviewers.
