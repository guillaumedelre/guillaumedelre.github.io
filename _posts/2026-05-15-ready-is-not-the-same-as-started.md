---
layout: post
title: "Ready Is Not the Same as Started"
date: 2026-05-15
categories: [development]
tags: [symfony, cloud, kubernetes, docker, 12factor]
description: "The entrypoint script that works perfectly in Docker Compose has five jobs. In Kubernetes, each of those jobs belongs somewhere else."
---

The rolling deploy looked clean. A new pod started. Kubernetes saw the healthcheck pass — `php -v` returned zero — and began routing traffic to the new container.

For the next forty seconds — out of a possible sixty — that container was polling for the database.

Requests that landed on it during that window got errors. Not many — the window was short — but enough to show up as noise in the monitoring. The kind of noise that gets dismissed as a transient network issue and filed nowhere. The deploy succeeded. The pod eventually became ready. The mechanism that caused it was still there, waiting for the next deploy.

The entrypoint script does five things before FrankenPHP starts: copy a version file, verify the vendor directory, wait up to sixty seconds for the database, run pending migrations, install assets and set filesystem permissions. In Docker Compose, this is invisible. In Kubernetes, the gap becomes traffic.

## The gap between started and ready

Kubernetes decides whether to send traffic to a pod by watching its readiness probe. A pod whose readiness probe passes receives requests. A pod whose readiness probe fails is removed from the load balancer rotation until it recovers. This is the mechanism that makes rolling deploys safe: Kubernetes doesn't cut over to a new pod until that pod says it's ready.

The compose.yaml defines a healthcheck on every service:

```yaml
healthcheck:
    test: [ "CMD", "php", "-v" ]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 10s
```

`php -v` succeeds the moment the PHP binary is present — which is true from the first millisecond of container life. The `start_period: 10s` gives ten seconds before checks begin. But the entrypoint polling loop runs for up to sixty seconds before FrankenPHP even starts. At second ten, the healthcheck passes. The application is still waiting for the database.

The Dockerfile has a better signal:

```dockerfile
HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
```

This checks the actual metrics endpoint that FrankenPHP exposes, with a sixty-second grace period to cover the polling loop. That's closer. But in Kubernetes, the `HEALTHCHECK` instruction is ignored entirely. Kubernetes uses its own probe configuration. Without explicit probe definitions in the Kubernetes manifests, there are no readiness checks — and a pod is considered ready the moment its container starts.

Which means: pod starts, entrypoint begins polling, Kubernetes routes traffic, application is not yet serving. Requests arrive at a container that isn't ready to handle them.

## Three signals, three questions

Kubernetes separates container lifecycle into three distinct questions, each with its own probe type:

**startupProbe** — "Has the application finished starting?" Fires repeatedly until it passes, then hands off to liveness. Prevents the liveness probe from killing a container that's legitimately slow to initialize. For a container whose entrypoint can take sixty seconds, this is the right tool.

**readinessProbe** — "Is the application ready to handle requests?" Fails and passes throughout the container's life. When it fails, the pod is removed from the load balancer. This is what makes a rolling deploy safe.

**livenessProbe** — "Is the application still alive?" If it fails, Kubernetes restarts the container. Meant to catch hung processes, not slow startups.

The sixty-second polling loop belongs in the startupProbe's patience, not in application code:

```yaml
startupProbe:
    httpGet:
        path: /metrics
        port: 2019
    failureThreshold: 12    # 12 attempts × 5s = 60s max
    periodSeconds: 5
```

Once the startupProbe passes, a readinessProbe on the same endpoint takes over — telling Kubernetes when the pod is safe to receive traffic — and a livenessProbe watches for hung processes. But the startupProbe is the one that absorbs the slow start. The entrypoint polling loop becomes redundant: if the database isn't up within the startup probe window, Kubernetes kills the container and tries again.

## The migration problem

The polling loop is the most visible issue, but the migrations create a subtler one.

With a rolling deploy and two replicas, Kubernetes starts a new pod while the old one still serves traffic. Both pods run the same entrypoint. Both reach `doctrine:migrations:migrate`.

Doctrine's migration table tracks which migrations have already executed, so a completed migration won't run twice. But if two pods start simultaneously and both see a pending migration, both attempt to run it at the same time. Whether that's safe depends on the migration: additive schema changes are usually fine; destructive ones less so. And you don't get to choose which ones run on a deploy that didn't expect to coordinate. `--all-or-nothing` wraps migrations in a transaction and rolls back everything if one fails — it's about atomicity within a single run, not coordination across processes.

The cleaner approach separates the two concerns into two init containers: one that waits for the database, one that runs migrations. The main container starts only after both complete:

```yaml
initContainers:
    - name: wait-for-db
      image: authentication:latest
      command: ["php", "bin/console", "dbal:run-sql", "-q", "SELECT 1"]
    - name: migrate
      image: authentication:latest
      command: ["php", "bin/console", "doctrine:migrations:migrate", "--no-interaction", "--all-or-nothing"]
```

Even with init containers, multiple pods starting simultaneously — initial deploy, after a node failure, or under autoscaling pressure — will each attempt to run migrations. Solving that properly — through a Helm pre-upgrade hook, a `maxSurge: 0` strategy, or a separate migration Job — is a topic in itself. What matters here is that the entrypoint is the wrong place to host that decision: it can't coordinate across pods, and it ties migration execution to application startup in a way that's hard to untangle later.

Factor XII of the <a href="https://12factor.net/admin-processes" target="_blank" rel="noopener noreferrer">twelve-factor methodology</a> — admin processes run in the same environment as the application — is satisfied either way. The question is whether "same environment" means "same entrypoint script" or "same image, separate process". In Kubernetes, the latter is safer.

## What the entrypoint's real job is

Strip out the database wait (now a startupProbe or init container), the migrations (now an init container or Job), and the assets install (a build-time operation that belongs in the Dockerfile), and the entrypoint has one remaining job: start the application.

```sh
exec docker-php-entrypoint "$@"
```

Factor IX of the twelve-factor app asks for fast startup and graceful shutdown. A container whose startup takes sixty seconds because it's waiting for external dependencies is not fast. It means rolling deploys are slow, recovery after a crash is slow, and horizontal scale-out creates a sixty-second gap before each new pod contributes.

Fast startup is not just a nice-to-have. It's what makes the rest of the cloud model work. When a pod can start in seconds, the orchestrator can scale aggressively and recover quickly. When it takes a minute, you add headroom everywhere — longer probe timeouts, larger deployment windows, more conservative scaling policies — and the system becomes rigid.

## The Docker Compose tax

The entrypoint accumulates these responsibilities for a reason. In Docker Compose, there is no init container concept. There is no startupProbe. Services declare `depends_on`, but without health conditions, that's just startup ordering — not readiness. The entrypoint fills the gap.

This is not a design flaw. It's a reasonable adaptation to the constraints of Docker Compose. The script works. It handles edge cases (the database timeout, unrecoverable errors, missing migrations directory). Someone tested it.

The issue is the assumption that the same script works equally well in Kubernetes. It runs. The application eventually starts. But it bypasses the probe system that makes Kubernetes deployments reliable, and it puts migration responsibility in a place where coordination across pods is difficult to reason about.

The series of migrations this codebase went through — [cache adapters](/2026/05/15/the-cache-that-was-lying-to-us/), [log handlers](/2026/05/15/no-witnesses/), [image secrets](/2026/05/15/layers-remember-everything/), [scheduler coordination](/2026/05/15/the-job-that-never-exits/), [media storage](/2026/05/15/three-adapters-one-variable/) — all of them were changes to application code or configuration. This one is different. It requires the infrastructure to gain awareness of what "ready" means for this application, and it requires the entrypoint to give up responsibilities it currently owns.

That's a harder conversation. But the startupProbe is waiting for it.

---

**This series on migrating a Symfony platform to Kubernetes:**

1. [The Cache That Was Lying to Us](/2026/05/15/the-cache-that-was-lying-to-us/) — Factors VI, VIII, IV
2. [No Witnesses](/2026/05/15/no-witnesses/) — Factors XI, IX
3. [Layers Remember Everything](/2026/05/15/layers-remember-everything/) — Factors III, V
4. [The Job That Never Exits](/2026/05/15/the-job-that-never-exits/) — Factors XII, IX
5. [Three Adapters, One Variable](/2026/05/15/three-adapters-one-variable/) — Factors IV, VI
6. **Ready Is Not the Same as Started** — Factors IX, XII *(you are here)*
