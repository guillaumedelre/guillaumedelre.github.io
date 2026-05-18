---
title: "What Survives the Build"
date: 2026-05-14T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 2
categories: [development]
tags: [symfony, cloud, docker, kubernetes, 12factor, security, secrets]
description: "How committed .env files feed a build step that bakes credentials into Docker layers — and what it takes to empty the file down to four lines."
---

At some point during a cloud migration audit, someone ran this:

```bash
docker run --rm <image> php -r "var_dump(require '.env.local.php');"
```

The output showed everything that `composer dump-env prod` had compiled into the image at build time. Which meant it showed everything that had been in the `.env` file when the image was built. Which meant it showed these, among others:

```dotenv
INFLUXDB_INIT_ADMIN_TOKEN=<influxdb-admin-token>
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=admin123
BLACKFIRE_CLIENT_ID=<blackfire-client-id>
BLACKFIRE_CLIENT_TOKEN=<blackfire-client-token>
BLACKFIRE_SERVER_ID=<blackfire-server-id>
BLACKFIRE_SERVER_TOKEN=<blackfire-server-token>
NGROK_AUTHTOKEN=replace-me-optionnal
```

Twenty-five variables in total. Every credential that had accumulated in the root `.env` over three years, now permanent in an image layer.

## How `dump-env` works

`composer dump-env prod` is a legitimate Symfony optimization. Instead of parsing `.env` files on every request, the runtime loads a pre-compiled PHP array from `.env.local.php`. Faster and simpler.

The problem is what it reads. The Dockerfile copies the repository into the image with `COPY . ./`, `.env` included. Then `dump-env prod` reads that file and compiles every variable into `.env.local.php`. The image ships with a frozen snapshot of the credentials that were in `.env` at build time.

Docker layers are immutable archives. Even if a subsequent step removed `.env` from the container filesystem, the layer containing it would still exist inside the image. `docker save <image>` produces a tarball of every layer; extracting any file from any point in the build history is straightforward. The credentials are invisible at runtime. They are not gone.

[Factor V](https://12factor.net/build-release-run) calls this out directly: a build artifact should be environment-agnostic, with config arriving at the release step from outside. Once credentials are compiled in, the image is no longer portable. You can't promote it across environments. You build twice and hope the second build behaves like the first.

## How twenty-five variables accumulate

Before tracing how this gets fixed, it's worth understanding how it happened.

The `BLACKFIRE_*` tokens are the easy case to understand. A team member sets up profiling, needs to share the configuration, and the repository is already open to everyone. One line in `.env` is the path of least resistance. The InfluxDB and Grafana credentials follow the same logic — shared tooling, shared repo, one commit.

Then there are the variables that reveal a different kind of drift. In some of the service-level `.env` files:

```dotenv
APP__RATINGS__SERIALS='{"brand1":{"fr":"12345"},...}'  # ~40 lines of JSON
APP__YOUTUBE__CREDENTIALS='{"brand1":{"client_id":"xxx","refresh_token":"yyy"},...}'
```

Audience measurement serial numbers. YouTube API refresh tokens per brand. These aren't secrets in the Blackfire sense. They're business data — the kind of values that vary between brands and environments, that someone decided to version in `.env` because they behaved like configuration and `.env` was where configuration lived.

Twenty-five variables is the sum of incremental decisions, none of which felt wrong in isolation. The problem is structural: when `.env` is the only answer available, everything starts looking like it belongs there.

## Where things actually belong

Emptying the file required answering one question for each variable: *where does this actually belong?*

The answers revealed three categories that the team had never explicitly named:

**Static config** lives in code. Business rules, routing logic, Symfony parameter files — anything that doesn't vary between deployments. A change requires a rebuild. The JSON blobs for audience measurement serials turned out not to be static config at all: they were queried from a dedicated Config service at runtime. They had no business being in a file.

**Environment config** varies between deployments: hostnames, connection strings, third-party credentials. This is what [Factor III](https://12factor.net/config) means by "config in environment variables" — real OS-level variables injected by the runtime, never files that travel with the code. In Kubernetes, this becomes a ConfigMap for non-sensitive values and a Kubernetes Secret for credentials. The choice for secrets management was SOPS — credentials are encrypted and committed to git, rather than stored in an external vault like Azure Key Vault or HashiCorp Vault. A vault trades simplicity for auditability: automatic rotation, centralized audit logs, workload identity-based access with no key to protect. SOPS trades those capabilities for a simpler operational model — no external service to query at deploy time, secrets travel through the normal code review process, git history serves as the audit trail. The accepted downsides are manual rotation and the responsibility of protecting the decryption key itself. For the team's scale, the tradeoff was deliberate.

**Dynamic config** changes without a deployment: editorial parameters, per-brand thresholds, content moderation settings. It belongs in a database, managed through the application's Config service. Some of what had accumulated in `.env` files was this category all along, passing as static defaults because it changed rarely enough that nobody noticed.

Once the categories had names, the variables sorted themselves. The root `.env` ended at four lines:

```dotenv
DOMAIN=platform.127.0.0.1.sslip.io
XDEBUG_MODE=off
SERVER_NAME=:80
APP_ENV=dev
```

Safe defaults. Nothing sensitive. `dump-env prod` now compiles empty strings; real values arrive at runtime from Kubernetes.

## The PostgreSQL image

The PostgreSQL image used in CI has a hardcoded password:

```dockerfile
FROM postgres:15
ENV POSTGRES_PASSWORD=admin123
```

This looks like the same problem. It isn't, because the threat model is different. The CI database is ephemeral — it exists for the duration of a pipeline run, contains no real data, and runs in an isolated network. A hardcoded password on a throwaway test database is an acceptable risk, not a policy exception.

In production, the question doesn't arise: the platform uses Azure Flexible Server, a managed PostgreSQL service. There is no Docker image. Credentials arrive via Helm chart injection, never touching a layer.

## What survives the build now

The image that ships to production now contains a guarantee: `var_dump(require '.env.local.php')` returns only empty strings and safe defaults. The credentials aren't there because they were never put there — they arrive at runtime, from outside.

That's the responsibility boundary `dump-env` had been quietly erasing: the image is the application, the runtime is the environment. They should not know each other's secrets.
