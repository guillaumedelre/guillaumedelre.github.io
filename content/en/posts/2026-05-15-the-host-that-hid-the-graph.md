---
title: "The Host That Hid the Graph"
date: 2026-05-15T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 4
categories: [development]
tags: [symfony, cloud, kubernetes, 12factor, microservices, http-client]
description: "Thirteen services sharing six identical gateway variables. The config looked simple. The dependency graph was invisible — until Kubernetes asked where each service actually lived."
---

Every service in the platform had these six variables:

```bash
APP__GATEWAY__PRIVATE__HOST="platform.internal"
APP__GATEWAY__PRIVATE__PORT=80
APP__GATEWAY__PRIVATE__SCHEME="http"
APP__GATEWAY__PUBLIC__HOST="platform.internal"
APP__GATEWAY__PUBLIC__PORT=80
APP__GATEWAY__PUBLIC__SCHEME="http"
```

Thirteen services, six variables each, one value. Reading any service's configuration, the architecture looked flat. Everything talked to the same host. That was the whole picture.

It wasn't.

## How the gateway worked

The gateway sat in front of every service and handled all inter-service traffic. A service calling the content API would construct a request to `http://platform.internal/content/api/` — the gateway received it, identified the target from the URL path, and forwarded it to the right backend. Every inter-service HTTP client in `framework.yaml` followed the same pattern:

```yaml
content.client:
    base_uri: "%http_client.gateway.base_uri%/content/api/"
    headers:
        Host: "%env(APP__GATEWAY__PRIVATE__HOST)%"
```

The `http_client.gateway.base_uri` parameter was assembled from the GATEWAY vars. The gateway knew where each service ran. The services didn't need to know. From their perspective, everything was `platform.internal`.

This worked. For years, it worked well. Adding a service meant adding one DNS alias in the gateway config, not touching thirteen `.env` files. The gateway abstracted the topology. The services stayed decoupled from the infrastructure detail of who ran where.

## What the gateway was absorbing

The abstraction had a cost that didn't show up until you tried to read the system.

Looking at `content`'s env file, you saw six gateway variables and nothing else about inter-service communication. To find out that `content` called `conversion`, `shorty`, and `media`, you had to read `framework.yaml`. To find out that `pilot` called ten external services, you had to trace through the HTTP clients one by one and count.

The number was ten. Authentication, bam, config, content, conversion, media, product, shorty, sitemap, social. Ten of the platform's thirteen services that `pilot` depended on at runtime, none of them visible from its configuration. Six variables said: talk to the gateway. They said nothing about the shape of what lay behind it.

That information existed — in the code, in the framework config, in the heads of the people who had built those integrations. It just didn't live anywhere you could read at a glance.

## What Kubernetes made explicit

On-premise, the gateway was a single resolvable hostname. One DNS record, one set of variables, one place to update. Kubernetes doesn't work that way. Each service gets its own DNS name inside the cluster — `content.namespace.svc.cluster.local`, `conversion.namespace.svc.cluster.local`. Inter-service traffic goes directly, service to service, not through a shared gateway.

Moving to Kubernetes meant the gateway abstraction had to give way. Each service needed to know, concretely, where each of its dependencies lived. The six generic variables couldn't express that.

The refactor replaced them with per-target HOST variables — one per service dependency, named for the target:

```bash
# content/.env — content calls these four services
APP__CONFIG__HOST="platform.internal"
APP__CONVERSION__HOST="platform.internal"
APP__MEDIA__HOST="platform.internal"
APP__SHORTY__HOST="platform.internal"
```

```bash
# pilot/.env — ten service dependencies
APP__AUTHENTICATION__HOST="platform.internal"
APP__BAM__HOST="platform.internal"
APP__CONFIG__HOST="platform.internal"
APP__CONTENT__HOST="platform.internal"
APP__CONVERSION__HOST="platform.internal"
APP__MEDIA__HOST="platform.internal"
APP__PRODUCT__HOST="platform.internal"
APP__SHORTY__HOST="platform.internal"
APP__SITEMAP__HOST="platform.internal"
APP__SOCIAL__HOST="platform.internal"
```

Each HTTP client in `framework.yaml` got its own `base_uri` built from its target's HOST variable, and the `Host` header gave way to a `User-Agent` that identified the caller:

```yaml
content.client:
    base_uri: "%env(APP__HTTP__SCHEME)%://%env(APP__CONTENT__HOST)%:%env(APP__HTTP__PORT)%/content/api/"
    headers:
        User-Agent: "Platform Content - %semver%"
```

The change isn't cosmetic. In the old setup, the explicit `Host` header ensured requests reached the correct gateway virtual host regardless of URL resolution. In the new setup, each client points directly at its target's DNS name — the right `Host` is derived from the `base_uri` automatically. The header slot doesn't go empty: `User-Agent` now identifies the calling service, which surfaces in logs and distributed traces without any additional instrumentation.

## The discomfort of legibility

`pilot`'s env file went from nine gateway variables to ten service-specific HOST variables. The file got longer. The architecture didn't get simpler — the ten dependencies were there before and they're still there now. What changed is that they're readable.

[Factor III](https://12factor.net/config) says to store configuration in the environment. The old approach satisfied that literally: six variables, all in env files, none hardcoded. But variables that collapse the entire dependency graph into a single opaque hostname aren't really configuration — they're a shorthand that trades legibility for convenience. Factor III doesn't ask only that configuration be externalized — it implicitly assumes the externalized configuration remains informative.

The refactor didn't simplify anything. It made the complexity visible. `pilot`'s ten HOST variables document, in the `.env` file itself, the ten services it depends on. A new team member reading that file learns something real about the architecture. The old file taught them that there was a gateway.

There's a version of this story where you read the final state and conclude the team did unnecessary work — they replaced six variables with ten, all pointing at the same host anyway. In local development, `platform.internal` still resolves to the same place. The functional behavior didn't change.

The change is in what the configuration communicates. In Kubernetes, the HOST values diverge: each target gets its own cluster-internal DNS name, different per environment. The variables now carry real information. The refactor prepared the config to be honest about a topology it had been quietly simplifying for years.
