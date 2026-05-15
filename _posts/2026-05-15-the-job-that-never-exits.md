---
layout: post
title: "The Job That Never Exits"
date: 2026-05-15
categories: [development]
tags: [symfony, cloud, kubernetes, 12factor, messenger, scheduler]
description: "Symfony Scheduler versus Kubernetes CronJob: not a choice between two tools, but a choice of where coordination should live."
---

The catalog listed ten of them. Containers with names like `authentication_cron_purge`, `content_cron_youtube`, `sitemap_cron_generation`. Each one running the same kind of command:

```bash
php bin/console messenger:consume scheduler_purge
```

In Docker Compose, this works exactly as it looks: a container starts, runs `messenger:consume`, and stays alive indefinitely. The Symfony Scheduler fires tasks at the configured times, the consumer picks them up, they run. The container never exits unless told to.

Moving this to Kubernetes raises a question that Docker Compose never asked: where should the scheduling coordination live?

## Two places to put the same logic

Kubernetes has a native answer for scheduled work: the CronJob. It reads a cron expression from a manifest, spins up a Pod at the right time, and when the command exits, the Pod terminates. The schedule lives in the infrastructure. The container is ephemeral.

```yaml
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: purge
              command: ["php", "bin/console", "fmm:purge:refresh-token"]
          restartPolicy: OnFailure
```

The Symfony Scheduler puts the same logic in PHP. The schedule is a class. The cron expression is a string in code. The coordination happens in the application layer, not in the infrastructure manifest.

Neither approach is wrong. They answer different questions. The choice is about where you want the coordination to live — and what guarantees that location can offer.

## The lock that changes everything

Each schedule class in the platform defines its tasks in code:

```php
#[AsSchedule(name: 'purge')]
final readonly class PurgeSchedule implements ScheduleProviderInterface
{
    public function __construct(
        private LockFactory $lockFactory,
    ) {}

    public function getSchedule(): Schedule
    {
        $schedule = new Schedule();

        $schedule->add(RecurringMessage::cron('0 0 * * *',
            new RunCommandMessage('fmm:purge:refresh-token')
        ));
        $schedule->add(RecurringMessage::cron('0 0 * * *',
            new RunCommandMessage('fmm:purge:session-log --days=60')
        ));

        $schedule->lock($this->lockFactory->createLock('schedule_purge'));

        return $schedule;
    }
}
```

The last line is the argument. `$schedule->lock()` acquires a Redis-backed distributed lock before processing any scheduled message. If the authentication service runs three pods, all three run `messenger:consume scheduler_purge`. All three see the scheduled task at midnight. Only one acquires the lock. The other two skip it.

This is coordination in the application layer — visible in the code, tested with the rest of the application, backed by the same Redis instance already used for the cache and session handling.

Kubernetes CronJobs push this coordination to the infrastructure layer, but they don't eliminate it. The CronJob controller guarantees *at least once* execution — under certain failure conditions (controller restarts, clock skew, concurrent triggering), a CronJob can fire twice. The application still needs to handle idempotency. With `LockFactory`, that requirement is explicit and in the same codebase as everything else.

## What Factor XII actually covers

Factor XII of the <a href="https://12factor.net/admin-processes" target="_blank" rel="noopener noreferrer">twelve-factor methodology</a> is about admin processes: one-off tasks that run in the same environment as the application, then exit. Migrations, data fixes, manual re-indexing. The principle is that these tasks use the same image and the same config as the running application, not a separate environment set up by hand.

The platform follows this for database migrations: `doctrine:migrations:migrate` runs in the container entrypoint, using the same image as the application. Same environment variables, same database connection, same Symfony kernel. When it finishes, the application starts.

Recurring scheduled tasks are a different category. They don't exit. They coordinate across instances. They're closer to long-running workers than to one-off admin commands — and that's exactly the model Symfony Scheduler uses. The connection to Factor XII is that both patterns run in the same environment as the application. Where they differ is in shape: one-shot versus persistent.

## The container that waits

A `messenger:consume scheduler_purge` container running at noon is doing nothing. It's listening on a Messenger transport, waiting for the scheduler to emit a message at midnight. That message will trigger the purge command. Then it will wait again.

The natural objection is cost: ten containers running continuously to fire on schedule, sometimes once a day, sometimes more often. This is not waste in the cloud-native sense — it's the model. A long-running worker that coordinates through a shared lock is the same pattern as any queue consumer. The container is idle most of the time. The lock ensures that when the moment comes, exactly one instance handles it.

What the current setup is missing is the signal to Kubernetes about how long to wait on shutdown. Symfony Messenger workers are designed to handle SIGTERM gracefully — they finish the current message before exiting, provided the handler itself doesn't block. But Kubernetes defaults to thirty seconds before sending SIGKILL. A purge job that takes two minutes would be interrupted. The fix belongs in the Deployment spec:

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 300  # match the longest expected task
```

Set it to something longer than the worst-case task duration. The scheduler consumer finishes its work. Kubernetes waits. No job gets cut short mid-run.

## The ten containers

Across the platform, ten scheduler consumers run in Docker Compose and would become Deployments in Kubernetes:

| Container | Schedule |
|---|---|
| `authentication_cron_purge` | midnight daily |
| `authentication_cron_ldap` | midnight + 2min daily |
| `bam_cron_purge` | midnight daily |
| `config_cron_redis` | env-driven |
| `content_cron_applenews` | env-driven |
| `content_cron_liveblog` | env-driven |
| `content_cron_performance` | env-driven |
| `content_cron_publication` | env-driven |
| `content_cron_purge` | midnight daily |
| `content_cron_youtube` | env-driven |

Six of the ten use environment-driven cron expressions, letting the schedule vary between staging and production without rebuilding the image — a Factor III concern that comes for free when the schedule lives in the application layer.

The migration plan for Kubernetes keeps all ten as Deployments. Each gets a `terminationGracePeriodSeconds` matching its worst-case duration. Each already has the Redis lock. The coordination that made this platform survive multi-pod deployments in Docker Compose is the same coordination that will make it correct in Kubernetes — because it was never in the infrastructure to begin with.

That is what the choice was really about.

---

**This series on migrating a Symfony platform to Kubernetes:**

1. [The Cache That Was Lying to Us](/2026/05/15/the-cache-that-was-lying-to-us/) — Factors VI, VIII, IV
2. [No Witnesses](/2026/05/15/no-witnesses/) — Factors XI, IX
3. [Layers Remember Everything](/2026/05/15/layers-remember-everything/) — Factors III, V
4. **The Job That Never Exits** — Factors XII, IX *(you are here)*
5. [Three Adapters, One Variable](/2026/05/15/three-adapters-one-variable/) — Factors IV, VI
6. [Ready Is Not the Same as Started](/2026/05/15/ready-is-not-the-same-as-started/) — Factors IX, XII
