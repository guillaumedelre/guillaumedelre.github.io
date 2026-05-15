---
layout: post
title: "No Witnesses"
date: 2026-05-15
categories: [development]
tags: [symfony, cloud, monolog, kubernetes, 12factor, observability]
description: "A service crashed in production and left no logs behind. Here is why fingers_crossed and cloud deployments do not mix well."
---

The service had crashed. We had the alert. We had the timestamp down to the second. We had <a href="https://grafana.com/oss/loki/" target="_blank" rel="noopener noreferrer">Loki</a> open and a query ready.

What we didn't have was any logs from the five minutes before the crash.

Promtail was running. It was healthy. It had been collecting logs from every other service without issue. But for this one, in the window that mattered, there was nothing. The service had crashed without leaving a trace.

## The setup that looked correct

The logging stack was reasonable. Each service wrote structured JSON to stdout using Monolog's logstash formatter:

```yaml
stdout:
    type: stream
    path: "php://stdout"
    level: "%env(MONOLOG_LEVEL__DEFAULT)%"
    formatter: 'monolog.formatter.logstash'
```

<a href="https://grafana.com/docs/loki/latest/" target="_blank" rel="noopener noreferrer">Promtail</a> collected container output via the Docker socket, parsed the JSON, extracted labels, pushed to Loki:

```yaml
scrape_configs:
    - job_name: docker
      docker_sd_configs:
          - host: unix:///var/run/docker.sock
      pipeline_stages:
          - json:
              expressions:
                  level: level
                  service: service
```

In theory: logs go to stdout, Promtail reads stdout, logs appear in Loki. The <a href="https://12factor.net/logs" target="_blank" rel="noopener noreferrer">twelve-factor app methodology</a> describes exactly this model for Factor XI — treat logs as event streams, write to stdout, let the environment handle collection and routing.

The application had stdout. Promtail was reading stdout. What could go wrong.

## What fingers_crossed takes with it

In production, the `when@prod` block replaced the simple `stream` handler with something more sophisticated:

```yaml
when@prod:
    monolog:
        handlers:
            main:
                type: fingers_crossed
                action_level: error
                handler: main_group
                excluded_http_codes: [404]
```

The `excluded_http_codes: [404]` line is itself a tell: without it, every 404 from a scanner or crawler triggers a full buffer flush, dumping megabytes of debug logs for malformed URLs. Someone had already learned that the hard way.

`fingers_crossed` is a well-known Monolog pattern. The idea is elegant: don't flood production logs with debug noise, but if something goes wrong, retroactively show what happened before the error. The handler buffers every log record in memory. The moment it sees an `error`, it flushes the entire buffer to the nested handler — giving you the full context leading up to the failure.

The problem is what happens when the failure isn't a logged error. It's an OOM kill. A SIGKILL from the orchestrator. A segfault. A process that stops responding and gets forcibly terminated.

In those cases, `fingers_crossed` never reaches its `action_level`. The buffer exists, full of the last five minutes of activity, and it vanishes with the process. The logs were there. They were in memory. They died before reaching stdout.

Factor IX of the twelve-factor app talks about disposability: processes should start fast and stop gracefully. On a clean shutdown (SIGTERM), a well-behaved process finishes its current work and exits. But crashes are not clean shutdowns, and memory buffers are not crash-safe. The service had been disposable in the sense that we could restart it; it was not disposable in the sense that its exit was transparent.

## The files nobody was reading

There was a second problem, quieter but just as persistent.

Every service had a `main_group` handler that routed logs to two destinations in parallel:

```yaml
main_group:
    type: group
    members: [main_file, stdout]

main_file:
    type: stream
    path: "%kernel.logs_dir%/%kernel.environment%.log"
    formatter: "monolog.formatter.logstash"
```

`var/log/prod.log` was being written on every service, in every environment, including production. The same content that went to stdout also went to a file inside the container. The file grew without rotation. The file was not accessible to Promtail (which read from the Docker socket, not from the container filesystem). The file consumed disk space. Nobody was reading it.

The audit channel was worse:

```yaml
audit_file:
    type: stream
    path: "%kernel.logs_dir%/audit.log"
    formatter: 'monolog.formatter.line'

audit:
    type: group
    members: [audit_file, stderr]
    channels: ['audit']
```

Audit logs went to `stderr` (visible to Promtail) and to `audit.log` (not visible to Promtail). The format in the file was a plain line format, not the structured JSON that Promtail expected. In practice, the audit trail existed in two places: one queryable, one buried in a container directory that survived only as long as the container did.

## What Factor XI actually requires

The eleventh factor is direct about this: an app should not concern itself with routing or storage of its output stream. It writes to stdout. Everything else is the environment's job.

That means no file handlers in production. Not as a backup. Not for audit trails. Not "just in case". The moment an application starts managing files, it takes on responsibility for rotation, retention, disk space, and accessibility — none of which belong inside a container.

The fix for the file handlers is straightforward. In `when@prod`, remove every `*_file` handler and every group that includes one. The audit channel gets the same treatment: stderr only, structured JSON, no file:

```yaml
when@prod:
    monolog:
        handlers:
            stdout:
                type: stream
                path: "php://stdout"
                # defaults to "warning" — overridable per-deploy via env var for targeted debugging
                level: "%env(default:default_log_level:MONOLOG_LEVEL__DEFAULT)%"
                formatter: 'monolog.formatter.logstash'

            stderr:
                type: stream
                path: "php://stderr"
                level: error
                formatter: 'monolog.formatter.logstash'

            main:
                type: group
                members: [stdout]
                channels: ['!event', '!http_client', '!doctrine', '!deprecation', '!audit']

            audit:
                type: stream
                path: "php://stderr"
                level: debug
                formatter: 'monolog.formatter.logstash'
                channels: ['audit']
```

stdout for the main channel. stderr for errors and audit. Nothing else. Promtail picks up both via the Docker socket. The container writes nothing to disk. And audit logs are now structured JSON, queryable in Loki alongside everything else.

## The harder question about fingers_crossed

The file handlers were easy. `fingers_crossed` is more nuanced.

The pattern solves a real problem: in a busy production service, logging everything at debug level creates noise and cost. `fingers_crossed` lets you capture context without paying for it unless something actually goes wrong. It is a reasonable tradeoff when the failure mode you're protecting against is an application-level error (an exception, a 500, a slow query).

It is not a reasonable tradeoff when the failure mode is a process crash. And in a Kubernetes environment, process crashes happen: OOM evictions, liveness probe failures, node pressure. Exactly the cases where you most need the logs.

One approach: keep `fingers_crossed` but reduce the buffer size. By default it keeps everything since the last reset. Set `buffer_size: 50` and you cap memory usage, which also limits what gets lost on crash. You won't have the full context, but you'll have the last fifty records.

Another approach: accept that debug logs are expensive and raise the default log level in production. Then you don't need `fingers_crossed` at all — if info and above go directly to stdout, nothing is ever buffered.

The approach we landed on: drop `fingers_crossed`, raise the default level to `warning`, keep a debug override available via env var for targeted investigation. The logs we care about appear immediately. The ones we don't are never written. Nothing is buffered.

## Crashes don't flush

Factor XI and Factor IX meet at the same point: a process dying mid-request. [The previous article in this series](/2026/05/15/the-cache-that-was-lying-to-us/) described the illusion of a service that worked perfectly on one pod but quietly misbehaved on two. This is the same illusion, one layer up: a service that appeared to log correctly, until the moment it most needed to.

The rule for production Monolog is blunt: if it doesn't reach stdout or stderr before the process exits, it doesn't exist. A file handler inside a container is invisible to the log collector and dies with the pod. A `fingers_crossed` buffer is invisible to the log collector and dies with the process.

Production tends to create the conditions where you need logs the most — OOM pressure, cascading failures, bad deploys — and those are exactly the conditions where both of these patterns fail you simultaneously. Write to stdout, default to a level that doesn't require buffering, and make the override available for when you actually need to debug something. The logs will be there. They won't be waiting for an error threshold that never fires.

---

**This series on migrating a Symfony platform to Kubernetes:**

1. [The Cache That Was Lying to Us](/2026/05/15/the-cache-that-was-lying-to-us/) — Factors VI, VIII, IV
2. **No Witnesses** — Factors XI, IX *(you are here)*
3. [Layers Remember Everything](/2026/05/15/layers-remember-everything/) — Factors III, V
4. [The Job That Never Exits](/2026/05/15/the-job-that-never-exits/) — Factors XII, IX
5. [Three Adapters, One Variable](/2026/05/15/three-adapters-one-variable/) — Factors IV, VI
6. [Ready Is Not the Same as Started](/2026/05/15/ready-is-not-the-same-as-started/) — Factors IX, XII
