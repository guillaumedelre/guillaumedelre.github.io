---
layout: post
title: "Observability on FrankenPHP containers before the cloud migration was done"
date: 2025-06-07
categories: [devops]
tags: [frankenphp, prometheus, grafana, caddy, monitoring, devops]
description: "Moving 14 PHP microservices to the cloud meant needing observability before the migration was done, not after. FrankenPHP's Caddy layer made that possible with two lines of config."
---

When you run workloads on-premise, you can get away with almost no observability. You have SSH. You have `top`. You have someone who knows that the authentication service always spikes on Monday mornings. Institutional knowledge substitutes for instrumentation, and nobody budgets the time to replace it.

Then you migrate to the cloud. The institutional knowledge doesn't follow. The SSH access is gone or inconvenient. And for the first time, you're staring at fourteen FrankenPHP containers with no idea what they're actually doing.

That's the moment you need metrics. Not eventually. Before the migration is done.

## The problem with doing it properly

The correct way to instrument a PHP service for Prometheus: add a client library, write counters and histograms around what you care about, expose a `/metrics` route, update the scrape config. For one service, that's a reasonable afternoon. For fourteen services mid-migration, it's a multi-sprint project that competes with everything else that needs to move.

The calculation is awkward. You need metrics to trust that the migration is going well. But adding metrics to everything before the migration means the migration takes longer. And the longer it takes, the more you need metrics to know where you stand.

Something had to give.

## What FrankenPHP carries without announcing it

FrankenPHP is not a PHP runtime that happens to use <a href="https://caddyserver.com" target="_blank" rel="noopener noreferrer">Caddy</a> as its web server. The relationship is inverted: Caddy is the server, and PHP is a Caddy module. Every HTTP request flows through Caddy before it reaches application code.

Caddy ships with a Prometheus-compatible metrics endpoint built in. No plugin, no extra binary. Enable the admin API and it's there.

`CADDY_GLOBAL_OPTIONS` is a FrankenPHP environment variable that injects directives directly into Caddy's global configuration block. Two lines are enough:

```yaml
environment:
    CADDY_GLOBAL_OPTIONS: |
        admin 0.0.0.0:2019
        metrics
```

`admin 0.0.0.0:2019` binds the admin API to all network interfaces - the default is localhost-only, which is unreachable from a Prometheus container on the same network. `metrics` enables the endpoint.

After that, every container responds to `GET :2019/metrics` with a full Prometheus payload. Request counts labeled by status code, latency histograms, active connections. No route added to the application. No `composer require`. No Dockerfile change.

One environment variable, added to each service definition in a single commit. Fourteen scrape targets, all producing data.

## A usable picture in Grafana

The Prometheus scrape config lists every service by its container name:

```yaml
scrape_configs:
    - job_name: caddy
      metrics_path: /metrics
      static_configs:
          - targets:
              - authentication:2019
              - content:2019
              - media:2019
              # all 14 services
```

Grafana sits on top of Prometheus. The Caddy community dashboard gives you request rates, error rates, and latency percentiles per service, per endpoint, per status code. Within a day of the migration landing in the new environment, there was something meaningful to look at.

The data tier follows the same logic: exporters for PostgreSQL, Redis, and RabbitMQ scrape at the infrastructure level without touching application code. Community dashboards exist for all of them.

## What this baseline actually covers

The HTTP metrics from Caddy are web server metrics, not application metrics. They answer: is this service receiving traffic, is it returning errors, how fast is it responding. The kind of questions you ask when something is broken and you need to triage in the dark.

They don't answer: how many items were processed today, which background job is stuck, what is the business impact of this latency spike. For those you need application instrumentation, and that work still exists when you have specific things to measure.

But in a migration context, that distinction matters less than it sounds. The things that break during a cloud migration are mostly infrastructure problems: a service that can't reach its database, a memory limit that was set too low, a queue consumer that stopped picking up messages. Those are exactly the things the baseline covers.

Getting instrumentation right for business-level events can wait until the platform is stable. Getting enough visibility to know whether the migration succeeded cannot.
