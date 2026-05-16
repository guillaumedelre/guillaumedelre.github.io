---
title: "Layers Remember Everything"
date: 2026-05-15T12:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 3
categories: [development]
tags: [symfony, cloud, docker, kubernetes, 12factor, security]
description: "How dev credentials end up inside production Docker images, and why Factor III and V are harder to follow than they look."
---

The `.dockerignore` file had the right instinct:

```
.env.local
.env.*.local
.env.local.php
```

Someone had thought about this. They knew that `.env.local.php` — the compiled environment cache that Symfony generates — shouldn't travel from a developer's machine into a Docker image. So they excluded it from the build context.

What they didn't account for is that the build generates a new one.

## The chain

The production Dockerfile follows a multi-stage pattern that is, on its face, well-structured. A base stage installs PHP extensions and system dependencies. A prod stage compiles the application without dev dependencies, optimizes the autoloader, and runs `composer dump-env prod` before handing off to FrankenPHP:

```dockerfile
RUN set -eux; \
    composer install --no-cache --prefer-dist --no-dev --no-autoloader --no-scripts; \
    composer dump-autoload --classmap-authoritative --no-dev; \
    composer dump-env prod; \
    sync;
```

`composer dump-env prod` is a legitimate performance optimization. Instead of parsing `.env` files on every request, Symfony loads a pre-compiled PHP file — `.env.local.php` — that returns an array of key-value pairs directly. Fast, clean, idiomatic.

But it has to read something to compile. And what it reads is the `.env` file that was copied into the image a few steps earlier via `COPY --link . ./`.

That `.env` file is committed to the repository. In the `media` service, it contains:

```dotenv
APP__JWT_SECRET="w4kg7YL9Fk5zmzeapdYjQHyICT8G8JGt1JCoUS7j4Wf2JgczPu76wjKVrIZHRw7a"
APP__HTTP_CACHE__CDN_ACCESS_TOKEN="akab-65n77xfcazkbwtgy-zfakzr6xqak5ths6"
APP__HTTP_CACHE__CDN_CLIENT_SECRET="s+LMM+7QtotVhIdtHhyLjWC6fsYK8aQDUA8mb..."
APP__HTTP_CACHE__CDN_CLIENT_TOKEN="akab-nz3ctehn6l5bpz6f-ar2klnztmief3fq7"
```

These values are described as development credentials. But the Akamai token format is not something you generate locally. And a JWT secret committed to a repository stops being a secret the moment the first developer clones the project.

`dump-env prod` reads them, compiles them into `.env.local.php`, and the image ships with a cached copy of every credential that was in `.env` at build time. The `.dockerignore` that excluded the developer's local `.env.local.php` doesn't touch the one the build just created.

## What Factor V is actually about

The <a href="https://12factor.net/build-release-run" target="_blank" rel="noopener noreferrer">fifth factor</a> draws a hard line between three stages:

- **Build**: transform code into an executable bundle. The output is environment-agnostic.
- **Release**: combine the build with deployment-specific config.
- **Run**: execute the release.

The key word is environment-agnostic. A build artifact should be the same binary regardless of whether it's heading to staging or production. You tag it, you store it, you promote it. The config comes in at the Release stage, injected from outside — environment variables, secrets managers, Kubernetes Secrets.

The multi-stage Dockerfile here has Build and Run. It skips Release as a concept. Config doesn't come in from outside; it gets compiled in. If your CI injects production secrets at build time — to avoid a separate injection step later — those values are frozen inside the image layers, permanent and extractable. The image is no longer environment-agnostic. It's a staging artifact or a production artifact, not both.

## What Factor III is actually about

The <a href="https://12factor.net/config" target="_blank" rel="noopener noreferrer">third factor</a> is precise: config is everything that varies between deployments. It should live in environment variables — real OS-level variables, not files that happen to use the word "environment" in their name.

The platform had already understood this at the application layer. Every config key follows a strict convention — `APP__<CATEGORY>__<KEY>` — and the Symfony config reads from these variables using `%env(APP__...)%`. The structure is right.

The gap is in what "environment variable" means at the infrastructure level. Writing `DATABASE_PASSWORD=dave` in `.env` and committing it to git produces a file whose content can be read by anyone with repository access, anyone who runs `docker inspect`, or anyone who does:

```bash
docker run --rm <image> php -r "var_dump(require '.env.local.php');"
```

And even if you tried to clean up after yourself, Docker layers are immutable archives. Even if a subsequent `RUN rm .env` step removes the file from the running container's filesystem, the layer that contains it still exists inside the image. `docker save <image>` produces a tar archive of every layer; from there, extracting any file from any point in the build history is a matter of unpacking the right layer. The file is invisible at runtime. It is not gone.

A real environment variable, in the twelve-factor sense, is injected by the runtime — a Kubernetes Secret mounted as an env var, a secrets manager sidecar, a CI system that writes values to the container environment without them ever touching a file. The application never knows where the value came from. The image never contains it.

## The fix in practice

The change is smaller than it looks.

`.env` should contain only structural defaults: empty strings, placeholder values, development-safe fallbacks. The convention the platform already uses works perfectly for this:

```dotenv
APP__JWT_SECRET=""
APP__HTTP_CACHE__CDN_ACCESS_TOKEN=""
APP__HTTP_CACHE__CDN_CLIENT_SECRET=""
```

`dump-env prod` compiles these empty values into `.env.local.php`. At runtime, Kubernetes injects the real values as OS environment variables, which take precedence. The image is environment-agnostic. The credentials never touch a layer.

If you genuinely need a secret during the build itself — a token to authenticate against a private Composer repository, an SSH key for a private git dependency — BuildKit provides a clean solution that leaves no trace in layers:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=composer_token \
    COMPOSER_AUTH=$(cat /run/secrets/composer_token) composer install
```

```bash
docker build --secret id=composer_token,src=./token.txt .
```

The secret is available during the `RUN` step and disappears when it finishes. It never appears in `docker history`, never in any layer.

The `.dockerignore` already shows the team was thinking in the right direction. The instinct to exclude `.env.local.php` was correct — it just needed to be applied one step upstream: keep real credentials out of `.env`, and the compiled output takes care of itself.

## Build once, deploy anywhere

The goal of separating Build from Release is that a single image can be promoted through environments without rebuilding. You build once, you test that artifact, you deploy it to staging, you promote the same image to production. The image is the unit of trust.

That property breaks as soon as environment-specific values are compiled in. An image built with staging secrets is not the same artifact as one built with production secrets, even if the code is identical. You're not promoting the tested artifact anymore; you're building a new one and hoping it behaves the same way.

The first two articles in this series — [the filesystem cache](/2026/05/15/the-cache-that-was-lying-to-us/) and [the log buffer](/2026/05/15/no-witnesses/) — described processes that worked fine on one instance and fell apart when multiplied. This is the same pattern at the image level: it works fine when you're deploying to one environment, and the seams appear when you try to promote across them.

The image should be dumb about credentials. The infrastructure should be smart about injecting them. That's the division of responsibility Factor III and Factor V are pointing at — and `dump-env prod`, used correctly, is perfectly compatible with both.
