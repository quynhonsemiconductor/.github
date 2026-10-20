# Develop window: PostgreSQL 18.6 and Valkey 8.1

Runbook for moving the develop environments to PostgreSQL 18.6 (RDS major 18) and Valkey 8.1 (ElastiCache engine 8.1). The owner runs every AWS command with `--profile qnsc-admin`. The agents hold no AWS credentials.

Set these once in the owner's shell, and do not put flags in variables (zsh does not word-split them):

```
export AWS_PAGER="" AWS_REGION=ap-southeast-1
```

Every command below names `--profile qnsc-admin --region ap-southeast-1` explicitly, so a missing export is harmless.

## Scope

- Shared runtime-dev cache (infra #195): Valkey 7.2 to 8.1, in place. Stack path in the infra repo: `live/runtime-dev`.
- rova develop (rova #749): RDS 17 to 18, in place.
- opshub develop (opshub #436): RDS 17 to 18, in place.
- KB develop (qnsc-kb-backend #249): RDS 16 to 18, in place, plus the pgvector extension update (revision `20261019_85`).

Not in this window: KB prod. The KB prod plan on #249 would create a new RDS instance and 62 resources, and no KB prod database exists. Nothing here applies to KB prod, and no step assumes one.

Owner policy: downtime is acceptable on develop. A manual snapshot before every risky apply is mandatory.

## Order at a glance

1. Read-only checks (a).
2. Compose and CI PRs merged by the SA: rova #747, opshub #434, KB #246, .github #31 (c).
3. Applies in this order, each preceded by its own snapshot and count check: infra #195, rova #749, opshub #436, KB #249 (d).
4. Post-apply checks per environment (e).
5. Rollback per step (f).

**A merge is an apply.** The infra-apply workflows run on push to `main` (infra: `live/**` and `modules/**`; the product repos: `infra/live/**` and `infra/modules/**`). The develop tier has no reviewer gate, so the merge itself applies the develop stack. Only the production tier is gated. Merge one develop PR at a time, and wait for its apply run to finish before the next merge.

The plan-guard does not run on develop. ci #227 (merged 2026-10-19) runs the plan guard in the prod apply only. Develop runs the destructive guard only, and KB develop runs no guard until KB #257 lands. None of the develop plans touch a guarded resource type, so the guard does not refuse any develop apply in this window.

## (a) Read-only checks, owner

### Count source: the CI plan, not a local plan

The counts below come from the `plan / plan (infra/live/develop)` job on each PR's final head. Local plans lack the `TF_VAR` values CI passes, so they can differ. Do not count a local plan. Check the counts on the final head's CI plan before merging each PR. The apply step runs its own plan and does not show counts while it runs, so the CI plan is the only place to check them before the merge.

Expected counts, from the CI plans on the heads listed. **Anything else is a stop.**

| Stack | Head checked | Add | Change | Destroy | Allowed actions |
|---|---|---|---|---|---|
| infra `live/runtime-dev` | `65170cb0e` (re-check on final head) | 0 | 1 | 0 | Valkey engine 7.2 to 8.1, `apply_immediately` true |
| rova `infra/live/develop` | `613eba9d7` | 3 | 1 | 3 | RDS engine 17 to 18 in place; `rova-develop-pg17` replaced by `rova-develop-pg18`; ECS task definition replacements for `api` and `worker` |
| opshub `infra/live/develop` | `a8f71bc5a` | 3 | 3 | 3 | RDS engine 17 to 18 in place; `opshub-develop-pg17` replaced; ECS task definition replacements for `api` and `worker`; two Fluent Bit config objects updated in place |
| KB `infra/live/develop` | `54c5cd1e4` | 1 | 1 | 1 | RDS engine 16 to 18 in place; `qnsc-kb-develop-pg16` replaced by `qnsc-kb-develop-pg18` |

The "Allowed actions" column is the complete list. In any stack, a destroy or replace of anything not in its row is a stop. In particular, stop on any `aws_db_instance` replace, any `aws_secretsmanager_secret` destroy, any subnet group, DNS record, identity, or target group destroy.

### Read-only checks before the window

Upgrade targets for the source engine versions. Replace the version to match each instance (rova and opshub develop run 17.9, KB develop runs 16.13):

```
aws rds describe-db-engine-versions --profile qnsc-admin --region ap-southeast-1 --engine postgres --engine-version 16.13 --query 'DBEngineVersions[0].ValidUpgradeTarget[].EngineVersion'
aws rds describe-db-engine-versions --profile qnsc-admin --region ap-southeast-1 --engine postgres --engine-version 17.9 --query 'DBEngineVersions[0].ValidUpgradeTarget[].EngineVersion'
```

Expect a list that includes `18.6`. If it does not, stop.

Confirm the instance identifiers (the stack names them `<product>-<env>`):

```
aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --query 'DBInstances[].[DBInstanceIdentifier,EngineVersion,DBInstanceStatus]' --output table
```

## (b) Snapshots, owner, right before each apply

Create the scratch directory once, before the first redirect:

```
mkdir -p ~/dw
```

Take each snapshot right before its own apply, not all at the start, so a stop in one step does not leave a stale snapshot behind. Wait for each snapshot to reach `available` before starting its apply.

Save the instance's settings before the apply. The rollback in (f) needs them:

```
aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --query 'DBInstances[0]' > ~/dw/rova-develop-before.json
```

Repeat for `opshub-develop` and `qnsc-kb-develop`.

For each RDS instance, before its apply:

```
aws rds create-db-snapshot --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --db-snapshot-identifier rova-develop-pre-pg18-20261020
aws rds wait db-snapshot-available --profile qnsc-admin --region ap-southeast-1 --db-snapshot-identifier rova-develop-pre-pg18-20261020
```

Use `opshub-develop` and `opshub-develop-pre-pg18-20261020` for opshub, and `qnsc-kb-develop` and `qnsc-kb-develop-pre-pg18-20261020` for KB.

For the shared cache, before infra #195:

```
aws elasticache create-snapshot --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache --snapshot-name qnsc-runtime-dev-cache-pre-valkey81-20261020
until [ "$(aws elasticache describe-snapshots --profile qnsc-admin --region ap-southeast-1 --snapshot-name qnsc-runtime-dev-cache-pre-valkey81-20261020 --query 'Snapshots[0].SnapshotStatus' --output text)" = available ]; do sleep 60; done
```

Save the cache's settings before the infra #195 apply. The cache rollback in (f) needs every one of them, including the parameter group, encryption, KMS key, user groups and port:

```
aws elasticache describe-replication-groups --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache > ~/dw/qnsc-runtime-dev-cache-before.json
aws elasticache describe-cache-clusters --profile qnsc-admin --region ap-southeast-1 --cache-cluster-id qnsc-runtime-dev-cache-001 --show-cache-node-info > ~/dw/qnsc-runtime-dev-cache-001-before.json
```

Record the snapshot names in the window log. The snapshots stay until the window is verified good for a week.

## (c) Compose and CI PRs, SA merges

Merge in this order, each after its checks are green on the final head: rova #747, opshub #434, KB #246, and .github #31 at any time. These PRs have no apply. When a base branch is deleted, the merge script retargets the stacked develop PRs (rova #749, opshub #436, KB #249) to `main`. Their CI must be green on their final heads before their step in (d).

## (d) Applies, in order

Each step: take its snapshot (b), check the counts on the final head's CI plan (a), merge, watch the apply run until it finishes, run the checks in (e), then continue.

1. **infra #195.** Merge; the merge applies `live/runtime-dev` (develop tier). Expected downtime: every develop environment loses the shared cache for the engine upgrade. Measured downtime: owner records it.
2. **rova #749.** Only #749 applies; #747 has no apply. Merge #749; it applies the rova develop stack. Expected downtime: rova develop API and worker during the RDS major upgrade. Measured downtime: owner records it.
3. **opshub #436.** Same procedure. Expected downtime: opshub develop, same as rova. Measured downtime: owner records it.
4. **KB #249.** Same procedure, after revision `20261019_84` is applied on the KB develop database (it is). The apply runs the RDS upgrade. The migration `20261019_85` runs with the new deploy. Expected downtime: KB develop API, worker and search. Measured downtime: owner records it.

Wait-for-infra can time out, and the deploy can then run on PG16. For KB, the extension check in (e) is therefore mandatory, not optional.

Do not merge any ci PR during the window.

## (e) Post-apply checks, owner

### Each RDS instance

Wait until the instance is available, then check the engine and parameter group. Use the instance's identifier:

```
aws rds wait db-instance-available --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop
aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --query 'DBInstances[0].[EngineVersion,DBParameterGroups[0].DBParameterGroupName,DBInstanceStatus,PendingModifiedValues]'
```

Expect `18.x`, `rova-develop-pg18`, `available`, and no pending values. Repeat for `opshub-develop` (`opshub-develop-pg18`) and `qnsc-kb-develop` (`qnsc-kb-develop-pg18`).

### Valkey

```
aws elasticache describe-cache-clusters --profile qnsc-admin --region ap-southeast-1 --cache-cluster-id qnsc-runtime-dev-cache-001 --query 'CacheClusters[0].[EngineVersion,CacheClusterStatus]'
```

Expect `8.1.x` and `available`.

### Health

Replace the placeholders with the develop URLs:

- rova API and opshub API: `curl -fsS https://<develop-api-url>/readyz` returns 200. `/livez` returns 200.
- KB API: `curl -fsS https://<kb-develop-url>/health/ready` returns 200. `/livez` returns 200.

### Worker

The worker ECS service is at its desired count with no restart loop, and its log shows a normal periodic tick after the apply. For opshub, the Graph worker heartbeat is fresh.

### KB search (unaccent path)

Run the same query before the apply for the baseline, and record the count:

```
curl -fsS "https://<kb-develop-url>/api/v1/search?q=ha%20noi" | head -c 400
```

After the apply, expect results, and the count must match the baseline. An error that mentions `unaccent` is a stop.

### KB pgvector (mandatory)

With database access, run both queries on the KB develop database:

```
SELECT extversion FROM pg_extension WHERE extname = 'vector';
SELECT indisvalid FROM pg_index i JOIN pg_class c ON c.oid = i.indexrelid WHERE c.relname = 'ix_article_chunks_embedding_hnsw';
```

The first must return `0.8.2`, the second `t`. If the deploy ran on PG16 (see the note in (d)), this check fails and KB is rolled back per (f).

## (f) Rollback, per step

Neither RDS nor ElastiCache can downgrade an engine in place. Rollback restores from the snapshot taken in (b), so every write made after the snapshot is lost. Record the lost write window.

OpenTofu state has to be fixed by hand in both cases, because the restore creates a new resource that the state does not know. The revert PR then brings the code back in line with the restored resource. Run every `tofu` command from the owner's checkout of the product repo, with `AWS_PROFILE=qnsc-admin AWS_REGION=ap-southeast-1 AWS_PAGER=""`.

### RDS (rova, opshub, KB)

1. Turn deletion protection off on the upgraded instance, wait, then delete it. Keep a final snapshot of the failed instance:

```
aws rds modify-db-instance --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --no-deletion-protection --apply-immediately
until [ "$(aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --query 'DBInstances[0].DBInstanceStatus' --output text)" = available ]; do sleep 60; done
aws rds delete-db-instance --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --final-db-snapshot-identifier rova-develop-pg18-failed-final
until ! aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop >/dev/null 2>&1; do sleep 60; done
```

The last loop exits when the identifier no longer exists. Do not use `aws rds wait` on the identifier while it is being deleted.

2. Restore the snapshot under the original identifier, on the old parameter group. Take each `<...>` value from `~/dw/rova-develop-before.json`. The snapshot keeps its own engine version, so there is no engine flag:

```
aws rds restore-db-instance-from-db-snapshot --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --db-snapshot-identifier rova-develop-pre-pg18-20261020 --db-instance-class <DBInstanceClass> --db-subnet-group-name <DBSubnetGroup.DBSubnetGroupName> --vpc-security-group-ids <VpcSecurityGroups[].VpcSecurityGroupId> --db-parameter-group-name default.postgres17 --no-publicly-accessible --no-multi-az --manage-master-user-password
until [ "$(aws rds describe-db-instances --profile qnsc-admin --region ap-southeast-1 --db-instance-identifier rova-develop --query 'DBInstances[0].DBInstanceStatus' --output text)" = available ]; do sleep 60; done
```

Use `--multi-az` instead of `--no-multi-az` if the saved file has `MultiAZ` true. Use `default.postgres16` for KB. Use `--manage-master-user-password` only if the saved file has a `MasterUserSecret`. The restored instance has a new RDS-managed secret ARN, so anything that references it is replaced on the next plan. Opshub's and KB's values come from their own saved files.

3. Point OpenTofu's state at the restored instance. The address is the one in the develop plan (`module.stack.module.rds.aws_db_instance.this`), and the import ID is the original identifier:

```
tofu -chdir=infra/live/develop state rm 'module.stack.module.rds.aws_db_instance.this'
tofu -chdir=infra/live/develop import 'module.stack.module.rds.aws_db_instance.this' rova-develop
```

4. Open the revert PR: `engine_version = "17"` (16 for KB), the pg17 parameter group, and the flags back to `false`. Its CI plan must match these counts, and anything else is a stop:
   - add 1: the `rova-develop-pg17` parameter group, created before the instance moves to it
   - change 1: the instance's parameter group and any omitted settings that differ from the saved file (tofu fixes them in place on the revert apply)
   - destroy 1: the orphaned `rova-develop-pg18` parameter group
   - plus one add and one destroy for each ECS task definition that references the new secret ARN (`api`, `worker`, and `migrator` if present in the plan)

   The instance must not be `must be replaced`. Merge the revert PR only after its CI plan shows these counts. The merge applies it.

If step 2 fails, the upgraded instance is already deleted and its final snapshot (`rova-develop-pg18-failed-final`) is the fallback. Do not delete that snapshot.

### Shared cache

1. Delete the upgraded group and wait until it is gone. ElastiCache has no rename, so the identifier must be freed first:

```
aws elasticache delete-replication-group --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache
until ! aws elasticache describe-replication-groups --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache >/dev/null 2>&1; do sleep 60; done
```

2. Create a group with the same identifier from the snapshot. Copy each setting from `~/dw/qnsc-runtime-dev-cache-before.json` and `~/dw/qnsc-runtime-dev-cache-001-before.json`: the parameter group (a `valkey7` family group), transit and at-rest encryption, the KMS key, user groups, port, node type, subnet group, security groups, cluster count and failover settings. Wait until `available`:

```
aws elasticache create-replication-group --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache --replication-group-description "restored from the pre-8.1 snapshot" --engine valkey --engine-version 7.2 --snapshot-name qnsc-runtime-dev-cache-pre-valkey81-20261020 --cache-node-type <CacheNodeType> --cache-subnet-group-name <CacheSubnetGroupName> --security-group-ids <SecurityGroupIds> --num-cache-clusters <n> --cache-parameter-group-name <valkey7 group> --port <Port> --transit-encryption-enabled --at-rest-encryption-enabled --kms-key-id <KmsKeyId>
until [ "$(aws elasticache describe-replication-groups --profile qnsc-admin --region ap-southeast-1 --replication-group-id qnsc-runtime-dev-cache --query 'ReplicationGroups[0].Status' --output text)" = available ]; do sleep 60; done
```

Drop the encryption flags if the saved file shows them off. If the saved file shows an auth token, do not pass it on the command line. Put the full request in a JSON file with `--cli-input-json file://~/dw/restore.json`, with the file mode set to `600`, and delete the file after the restore.

3. Run these from the owner's checkout of the **infra** repo (its root holds `live/`), not a product repo. Import the restored group into the state, then open the revert PR. The address is `module.shared_cache[0].aws_elasticache_replication_group.this[0]`, and the import ID is `qnsc-runtime-dev-cache`:

```
tofu -chdir=live/runtime-dev state rm 'module.shared_cache[0].aws_elasticache_replication_group.this[0]'
tofu -chdir=live/runtime-dev import 'module.shared_cache[0].aws_elasticache_replication_group.this[0]' qnsc-runtime-dev-cache
```

The revert PR sets `node_engine_version = "7.2"` in `live/runtime-dev`. Its CI plan should show no add or destroy. If it shows a change, list each attribute in it before merging. Apps read the same endpoint, so they do not change.

### KB app deploy (step 4 only)

Migration `20261019_85` ships inside the KB app image. If the KB develop stack is rolled back to PG16 and the app still runs the image with `20261019_85`, the migration runs against PG16. Pin or revert the KB app first:

- revert the #249 merge with a PR that restores the KB develop stack, or
- redeploy the image from before #249 with the backend deploy workflow.

Confirm the deployed image has no `20261019_85` before the RDS restore, and confirm the alembic head afterwards.

## Later: KB prod and the prod upgrades

The prod upgrades (rova #750, opshub #437) are not part of this window. Their parameter-group replacements are gated by the prod plan guard (ci #227). Each prod apply needs an allow entry for the parameter group, in `.allow-data-destroy` in its stack directory, with a reason, and the entry is removed after the upgrade is verified. KB prod has no database today, so it has no upgrade step.
