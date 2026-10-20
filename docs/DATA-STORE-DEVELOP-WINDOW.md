# Develop window: PostgreSQL 18.6 and Valkey 8.1

Runbook for moving the develop environments to PostgreSQL 18.6 (RDS major 18) and Valkey 8.1 (ElastiCache engine 8.1). Owner runs every AWS command with `--profile qnsc-admin`; the agents hold no AWS credentials.

Scope:
- Shared runtime-dev cache (infra #195): Valkey 7.2 to 8.1, in place.
- rova develop (rova #749): RDS 17 to 18, in place.
- opshub develop (opshub #436): RDS 17 to 18, in place.
- KB develop (qnsc-kb-backend #249): RDS 16 to 18, in place, plus the pgvector extension update (revision `20261019_85`).

Owner policy: downtime is acceptable on develop. A manual snapshot before every risky apply is mandatory.

## Order at a glance

1. Read-only plan check (a).
2. Manual snapshots of the three develop databases and the shared cache (b).
3. Compose and CI PRs merged: #747 (rova), #434 (opshub), #246 (KB), .github #31 (SA merges) (c).
4. Applies, one at a time, in this order: infra #195, then rova #749, opshub #436, KB #249 (d).
5. Post-apply checks per environment (e).
6. Rollback per step (f).

**A merge is an apply.** The infra-apply workflows in infra, rova, opshub and KB run on push to `main` (infra: `infra/live/**` and `infra/modules/**`; the product repos: the same paths). The develop tier has no reviewer gate, so the merge itself applies the develop stack. Only the production tier is gated, and only by a manual dispatch plus the production environment reviewer. Do not merge a second develop PR until the first apply run has finished.

## (a) Read-only plans, owner, before any merge

Run these against the PR head, in a throwaway worktree, not in a checkout with local changes:

```
git -C ~/Desktop/qnsc/rova fetch origin
git -C ~/Desktop/qnsc/rova worktree add ~/dw/rova-749 origin/infra/rova-develop-rds-pg18
cd ~/dw/rova-749
AWS_PROFILE=qnsc-admin AWS_REGION=ap-southeast-1 AWS_PAGER="" tofu -chdir=infra/live/develop init -input=false
AWS_PROFILE=qnsc-admin AWS_REGION=ap-southeast-1 AWS_PAGER="" tofu -chdir=infra/live/develop plan -input=false -out=develop.tfplan
AWS_PROFILE=qnsc-admin AWS_REGION=ap-southeast-1 AWS_PAGER="" tofu -chdir=infra/live/develop show develop.tfplan
```

Repeat the same three commands for opshub (`origin/infra/opshub-develop-rds-pg18`), qnsc-kb-backend (`origin/infra/kb-develop-rds-pg18`) and infra (`origin/infra/runtime-dev-valkey-81`, with `-chdir=infra/live/runtime-dev`).

Expected counts, from the CI plans of 2026-10-19. The heads changed in the rebase, so the counts must be re-checked on the new heads. **Anything else is a stop.**

| Stack | Add | Change | Destroy | What the change must show |
|---|---|---|---|---|
| infra `runtime-dev` | 0 | 1 | 0 | Valkey engine 7.2 to 8.1, `apply_immediately` true |
| rova `develop` | 3 | 1 | 3 | RDS engine 17 to 18 in place; parameter group `rova-develop-pg17` replaced by `rova-develop-pg18`; ECS task definition replacements |
| opshub `develop` | 4 | 3 | 4 | RDS engine 17 to 18 in place; parameter group replaced; ECS task definition replacements; tunnel-token secret version replaced (expected after PR 418) |
| KB `develop` | 1 | 1 | 1 | RDS engine 16 to 18 in place; parameter group `qnsc-kb-develop-pg16` replaced by `qnsc-kb-develop-pg18` |

Stop if the instance is `must be replaced`, if any subnet group, DNS record, identity or secret is destroyed, or if the counts differ.

## (b) Snapshots first, owner

Confirm the instance identifiers (the stack names them `<product>-<env>`):

```
aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --query 'DBInstances[].[DBInstanceIdentifier,EngineVersion,DBInstanceStatus]' --output table
```

Create and wait for each snapshot:

```
aws rds create-db-snapshot --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --db-snapshot-identifier rova-develop-pre-pg18-20261020
aws rds create-db-snapshot --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier opshub-develop --db-snapshot-identifier opshub-develop-pre-pg18-20261020
aws rds create-db-snapshot --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier qnsc-kb-develop --db-snapshot-identifier qnsc-kb-develop-pre-pg18-20261020
aws rds wait db-snapshot-available --profile qnsc-admin --region ap-southeast-1 --db-snapshot-identifier rova-develop-pre-pg18-20261020
aws rds wait db-snapshot-available --profile qnsc-admin --region ap-southeast-1 --db-snapshot-identifier opshub-develop-pre-pg18-20261020
aws rds wait db-snapshot-available --profile qnsc-admin --region ap-southeast-1 --db-snapshot-identifier qnsc-kb-develop-pre-pg18-20261020
```

Snapshot the shared cache too (in-place engine upgrades cannot be reversed on ElastiCache):

```
aws elasticache create-snapshot --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache --snapshot-name qnsc-runtime-dev-cache-pre-valkey81-20261020
```

Record the snapshot names and the KB search baseline (step e) in the window log before continuing.

## (c) Compose and CI PRs, SA merges

Merge in this order, each after its checks are green on the rebased head: rova #747, opshub #434, KB #246, then .github #31 at any time. The stacked develop PRs (rova #749, opshub #436, KB #249) are retargeted to `main` by the merge script when their base branch is deleted. Their CI must be green on the new heads before step (d).

## (d) Applies, in order

Each step: start, watch the apply run's plan step, check the counts against (a), let the apply finish, then continue.

1. **infra #195, automatic on merge.** Applies the `runtime-dev` stack (develop tier). Expected downtime: every develop environment loses the shared cache for the engine upgrade. Measured downtime: owner records it.
2. **rova #747 then #749.** Merge #749, which applies the develop stack. Expected downtime: rova develop API and worker are unavailable during the RDS major upgrade. Measured downtime: owner records it.
3. **opshub #434 then #436.** Same procedure. Expected downtime: opshub develop, same as rova.
4. **KB #246 then #249.** Merge #249 once revision `20261019_84` is applied on the KB develop database (it is). The apply runs the RDS upgrade, then the migration `20261019_85` runs with the new deploy. Expected downtime: KB develop API, worker and search. Measured downtime: owner records it.

Do not merge `ci` #227 during this window. Its plan guard refuses parameter group replacements, and the develop plans replace parameter groups. Finish the window first.

If an apply run fails after the RDS modification starts, stop. Do not merge or re-run until the instance state is checked (e).

## (e) Post-apply checks, owner, per environment

Engine and parameter group (read-only):

```
aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --query 'DBInstances[0].[EngineVersion,DBParameterGroups[0].DBParameterGroupName,DBInstanceStatus,PendingModifiedValues]'
```

Expect `18.x`, `rova-develop-pg18`, `available`, and no pending values. Repeat for `opshub-develop` and `qnsc-kb-develop`.

Valkey (read-only):

```
aws elasticache describe-cache-clusters --profile qnsc-admin --region ap-southeast-1 --cache-cluster-id qnsc-runtime-dev-cache-001 --query 'CacheClusters[0].[EngineVersion,CacheClusterStatus]'
```

Expect `8.1.x` and `available`.

Health endpoints (replace the placeholders with the develop URLs):

- rova API and opshub API: `curl -fsS https://<develop-api-url>/readyz` returns 200. `/livez` returns 200.
- KB API: `curl -fsS https://<kb-develop-url>/health/ready` returns 200, `/livez` returns 200.

Worker heartbeat: the worker ECS service is at its desired count with no restart loop, and its log shows a normal periodic tick after the apply. For opshub, the Graph worker heartbeat must be fresh.

KB search, the unaccent path (run the same query before the apply for the baseline):

```
curl -fsS "https://<kb-develop-url>/api/v1/search?q=ha%20noi" | head -c 400
```

Expect results. The count must match the baseline, because the query depends on `immutable_unaccent`. An error that mentions `unaccent` is a stop.

KB pgvector (if you have database access): `SELECT extversion FROM pg_extension WHERE extname = 'vector';` returns `0.8.2`, and `SELECT indisvalid FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid WHERE c.relname = 'ix_article_chunks_embedding_hnsw';` returns `t`.

## (f) Rollback, per step

The major-version upgrade cannot be reversed in place on RDS, and ElastiCache does not downgrade engines. Rollback always means restoring from the snapshot taken in (b), so writes made after the snapshot are lost.

- **Shared cache:** create a replication group from `qnsc-runtime-dev-cache-pre-valkey81-20261020` on engine 7.2, then point the develop stacks at it. Cache contents are rebuilt by the apps.
- **RDS (rova, opshub, KB):** restore `<product>-develop-pre-pg18-20261020` to a new identifier with engine 17 or 16 and its original parameter group family. Point the stack at the new endpoint, revert the stack change in the repo, and plan again before any apply. Record the lost write window.
- **Migration 85 (KB):** the extension update is not reversible by the migration. Rollback is the snapshot restore above.

Keep the snapshots until the window is verified good for a week.
