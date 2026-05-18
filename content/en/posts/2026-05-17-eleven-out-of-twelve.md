---
title: "Eleven Out of Twelve"
date: 2026-05-17T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 8
categories: [development]
tags: [symfony, cloud, docker, kubernetes, 12factor, doctrine, migrations]
description: "Eleven factors resolved cleanly. The twelfth: Doctrine migrations in the entrypoint, waiting on a governance question that code alone can't answer."
---

The `composer.json` in each service had this in its `post-install-cmd` section:

```json
"post-install-cmd": [
    "bin/console cache:clear --env=prod",
    "bin/console doctrine:migrations:migrate --no-interaction"
]
```

`post-install-cmd` runs during `composer install`, which in the production Dockerfile runs during the image build. There is no database available during a Docker build. The migration command either failed silently, or connected to nothing, or was skipped by Doctrine when it couldn't find a schema to compare against. In any case, it didn't migrate anything.

This is a clean violation of [Factor XII](https://12factor.net/admin-processes): admin processes — migrations, one-off scripts, console tasks — should run in the same environment as the application, against the actual production data. Running them at build time inverts the relationship. The image shouldn't know about the database. The database should be there when the image needs it.

## The move to the entrypoint

The migration command moved from `composer.json` to `docker-entrypoint.sh`. The shift looks small on a diff. The implications are not.

The entrypoint runs when the container starts, not when the image is built. The database is reachable. The entrypoint waits for it — up to 60 seconds, one attempt per second — before doing anything:

```sh
ATTEMPTS_LEFT_TO_REACH_DATABASE=60
until [ $ATTEMPTS_LEFT_TO_REACH_DATABASE -eq 0 ] || \
  DATABASE_ERROR=$(php bin/console dbal:run-sql -q "SELECT 1" 2>&1); do
    sleep 1
    ATTEMPTS_LEFT_TO_REACH_DATABASE=$((ATTEMPTS_LEFT_TO_REACH_DATABASE - 1))
done

if [ $ATTEMPTS_LEFT_TO_REACH_DATABASE -eq 0 ]; then
    echo "$DATABASE_ERROR"
    exit 1
fi
```

If the database doesn't respond within 60 seconds, the container exits with an error and Kubernetes restarts it. Once the database is ready, the migration runs:

```sh
if [ "$( find ./migrations -iname '*.php' -print -quit )" ]; then
    php bin/console doctrine:migrations:migrate --no-interaction --all-or-nothing
fi
```

Two changes from the original command: `--all-or-nothing` ensures that if any migration in a batch fails, the entire batch rolls back. And the `find` guard skips the command entirely if there are no migration files — useful for services that don't use Doctrine migrations at all.

This is genuinely better. The database is present. The migration runs in the real environment. The `--all-or-nothing` flag adds atomicity that the build-time version never had.

## What it doesn't solve

Two pods redeploying simultaneously both run the entrypoint. Both reach the database. Both find pending migrations. Both call `doctrine:migrations:migrate`.

Doctrine has a locking mechanism: a `doctrine_migration_versions` table that records which migrations have run, and the command checks it before applying. Under normal conditions this is fine: the second pod finds the table up to date and exits cleanly. The real failure modes are more specific: a migration long enough that the database lock times out before it completes, letting a second runner start the same migration before the first has finished; or a pod that crashes mid-migration before recording the version in the table, leaving the schema in an applied-but-unregistered state that the next pod will try to apply again.

The team's position is explicit: a brief deployment downtime is acceptable. Application versions aren't necessarily forward-compatible with older schema versions, so running N and N+1 simultaneously against the same database isn't safe anyway. The deployment strategy is Recreate: all old pods are terminated before any new pods start. The migration runs on first startup, no overlap between versions. It works.

But "it works" and "it's the right architecture" are different answers.

## What would be different

[Factor XII](https://12factor.net/admin-processes) says admin processes should run in "one-off processes." A process that runs once, for a specific purpose, against the production environment. The entrypoint is not one-off — it runs every time a container starts, including restarts, scaling events, and Kubernetes node movements.

Three alternatives exist, each with a different answer to the question of ownership:

**A Kubernetes init container** runs before the main container starts, in the same pod. It could run the migration, exit, and let the main container start only after it succeeds. The migration is isolated from the application runtime. The downside: the init container is another image to build and maintain, and it runs on every pod start — so a 14-service platform starting simultaneously still has a potential race.

**A Kubernetes Job** runs once, on demand or triggered by a deployment pipeline. It can be made to run before any pods are updated — serial, isolated, with a clear success or failure signal. The race condition goes away. The complexity moves to the deployment process: the Job must complete before the Deployment rollout begins, and the CI pipeline must coordinate both.

**A Helm hook** is the same concept expressed declaratively in the Helm chart. A `pre-upgrade` hook runs the migration before the application pods are updated. It's the most idiomatic Kubernetes answer. It also means the Helm chart is now responsible for running migrations — a decision that belongs to whoever owns the chart.

That last sentence is why the entrypoint hasn't changed. Moving migrations out of the application means deciding that the deployment infrastructure — not the application itself — is responsible for the schema. It's a governance question as much as a technical one, and governance questions take longer to resolve than code changes.

## The honest end

The migration block in the entrypoint is two lines. Literally: the `if [ "$( find ./migrations... )" ]` guard, and the `php bin/console doctrine:migrations:migrate` that follows. Eleven other factors have clean resolutions. The cache moved to Redis. The logs go to stdout. The filesystem is an S3 bucket. The CI assembles production images from the same commit it tests. The secrets don't travel in image layers.

Factor XII has an answer. It's just not the final one.

The migrations run at startup, with a real database, with atomicity, with a bounded retry window. That's better than running at build time against nothing. Whether they eventually move to a Job or a Helm hook is a conversation about who owns the schema — a question that a `kubectl apply` can't answer.
