---
title: "The Cache That Was Lying to Us"
date: 2026-05-16T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 6
categories: [development]
tags: [symfony, cloud, redis, cache, kubernetes, 12factor]
description: "How a single config line blocked horizontal scaling across 13 Symfony microservices, and what the twelve-factor app had to say about it."
---

The first time we ran two replicas of the same Symfony service behind a load balancer, everything looked fine. Health checks passed. Traffic split cleanly. Response times were good.

Then someone noticed the rate limiter was acting strange. Hit the API five times, get blocked. Hit it five more times on the next request, get through. Depending on which pod answered, you were a different person.

That was the cache talking. One config line, replicated across thirteen services, was blocking horizontal scaling entirely.

## One config file, thirteen times

We were preparing a platform of thirteen Symfony microservices to move to Kubernetes. The stack was already in good shape: FrankenPHP for the HTTP server, multi-stage Dockerfiles, a GitLab CI that pushed tagged images to a cloud registry. The pieces were there. We just needed to verify nothing would break when we started scaling pods horizontally.

A good checklist for that kind of audit is the <a href="https://12factor.net" target="_blank" rel="noopener noreferrer">twelve-factor app methodology</a> — twelve principles for building software that runs cleanly in cloud environments. Most factors were already covered without us doing anything deliberate about it.

Factor VII (port binding) came for free. FrankenPHP embeds Caddy directly into the PHP process. The container exposes its own HTTP endpoint, no Apache or Nginx to bolt on. The image is self-contained, which is exactly what the factor requires:

```dockerfile
HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
```

Factor II (dependencies) was handled by `composer.json` and the Dockerfile extensions. Factor X (dev/prod parity) was covered enough for our scope: same image, same backing services locally and in CI, which is the part that actually matters for what we were auditing.

Then I got to Factor VI.

## The problem with "it works on one server"

Factor VI says processes must share nothing. Nothing written to disk between requests, nothing in local memory that another instance can't see. If you need to persist state, put it in a backing service — a database, a cache cluster, a queue. The process itself stays disposable.

I opened `authentication/config/packages/cache.yaml`. Then `content/config/packages/cache.yaml`. Then `media/config/packages/cache.yaml`.

```yaml
framework:
    cache:
        app: cache.adapter.filesystem
```

Thirteen services. Thirteen times, word for word.

Every instance of every service was writing its cache to the local filesystem. Which meant every pod had its own private cache, invisible to every other pod. When the load balancer sent a request to pod A, it got pod A's cached version of reality. Pod B had built its own. They might have been generated at different times, from different source data, or one of them might not have been built yet at all.

The rate limiter was the most visible symptom because it had a counter. But the same divergence affected every piece of data we were caching: serializer metadata, route collections, Doctrine result caches. Two users sending identical requests could get different responses depending on which node happened to pick up the connection.

## Redis was already there

This is the part that stings a little. Redis was already in the stack. Every service had it configured via SncRedisBundle:

```yaml
# config/packages/snc_redis.yaml — present on all 13 services
snc_redis:
    clients:
        default:
            type: 'phpredis'
            alias: 'default'
            dsn: '%env(IN_MEM_STORE__URI)%'
```

Factor IV of the twelve-factor app says backing services should be attached resources, interchangeable through configuration. Redis was exactly that: reachable via an environment variable, ready to be swapped for a managed instance in the cloud. The plumbing was done. We just weren't using it for the application cache.

Some services even had it right for specific pools. The rate limiter in the authentication service:

```yaml
pools:
    rate_limiter.cache:
        adapter: cache.adapter.redis
```

Which explains the inconsistency we saw first. The rate limit *count* went to Redis (shared across pods). The cache backing the rate limit *check* went to the filesystem (local to the pod). Two sources of truth, one invisible to the other.

The fix was one line per service:

```yaml
framework:
    cache:
        app: cache.adapter.redis
        default_redis_provider: snc_redis.default
```

Thirteen files. Thirteen identical changes. The kind of fix that makes you feel like you should have caught it earlier, except it's perfectly invisible when you're running a single instance.

## What needs to move to Redis

The filesystem cache violated Factor VI (processes carry local state they shouldn't) and Factor VIII (you can't scale out without sharing that state). They're the same problem seen from two angles: VI describes what's wrong, VIII describes what you can't do because of it.

With a shared cache backend, a second pod is safe. The two pods build the same cache, see the same invalidations, agree on the same rate limits. You can add a third pod under load and remove it when traffic drops. The orchestrator handles it; the application doesn't need to know.

Without it, horizontal scaling is a liability. More pods means more divergence, more "works on my machine" bugs that are impossible to reproduce locally because local only runs one container.

Sessions had the same problem — and potentially a worse one. Twelve of the thirteen services were using `session.storage.factory.native` — which writes sessions to the filesystem by default. A user whose request lands on pod A gets a session tied to pod A. Their next request goes to pod B. Session gone, they're logged out. Only one service had `RedisSessionHandler` configured.

The partial mitigation is that most of the platform runs stateless JWT-based APIs, so session usage is limited. But "limited" isn't "zero". The services that do create sessions — authentication flows, temporary state during OAuth handshakes — have a user-visible failure mode waiting for the second pod. Either those sessions get moved to Redis, or the code that creates them gets removed. Leaving them as-is is a decision that waits for the first user whose session disappears without explanation.

## The other kind of state

Redis fixes the cross-pod problem. FrankenPHP introduces a different one worth knowing about.

In the standard PHP-FPM model, each request forks a fresh process. Every in-memory object — every cached value, every computed result — dies with the response. The process is stateless by construction.

FrankenPHP has a worker mode that doesn't follow that model. In worker mode, a single PHP process boots once, loads the kernel, wires the container, and handles multiple successive requests without restarting. Request throughput improves: no autoloader cold start, no container rebuild per request, fewer allocations. The tradeoff is that the PHP process now has a lifecycle that spans requests.

For cache, this adds a wrinkle. An `array` adapter or APCu pool accumulates entries across requests on the same worker. A cache invalidation pushed to Redis reaches the other pods immediately — but doesn't clear what's sitting in a worker's in-process memory. Two requests on the same pod can see different things: one hits a warm in-memory entry, the next triggers a Redis fetch after the in-process entry expires.

The platform keeps worker mode disabled (`APP__WORKER_MODE__ENABLED=false`). It's available — the infrastructure is there, the flag is wired — but it's not active. The performance gain didn't justify the audit. Every cache pool would need to be verified against worker-mode semantics; every place where state leaks between requests would become a potential bug.

The conservative position: keep PHP stateless at the process level even when the runtime doesn't require it. Factor VI's shared-nothing principle applies not just to the filesystem — it applies to the process itself.

## What was already working

To be fair to the codebase: the Symfony Scheduler was already using Redis for distributed locks:

```php
$schedule->lock($this->lockFactory->createLock('schedule_purge'));
```

In a multi-pod environment, you don't want five instances running the same purge job simultaneously. The lock prevents it. Redis makes the lock visible across pods. Whoever wrote the scheduler knew exactly what they were doing.

The same reasoning just hadn't propagated to the cache configuration — probably because when you're running a single instance, `cache.adapter.filesystem` is invisible. It works, it's fast, it requires zero configuration. The problem only appears at two.

## The four questions

Factor VI catches most applications off guard during a cloud migration. Not because developers don't know about stateless processes — they usually do — but because the filesystem is always there, and the problem stays hidden until you try to run a second instance.

Before scaling a Symfony service horizontally, four questions are worth answering:

- Where does the application cache go? (`cache.adapter.filesystem` needs to become `cache.adapter.redis`)
- Where do sessions go? (`session.storage.factory.native` needs Redis — or remove sessions entirely if you're JWT-only)
- Does anything write to `var/` at runtime that another pod would need to read?
- Is anything in your code path that needs to be mutually exclusive across pods? (if yes, that's a job for the <a href="https://symfony.com/doc/current/components/lock.html" target="_blank" rel="noopener noreferrer">Symfony Lock component</a> backed by Redis, not a local mutex)

If the answers all point to shared backing services, you're ready. If any of them points to the local filesystem, production will find the pod that built its cache three hours ago and serve it to the user who least expects it.
