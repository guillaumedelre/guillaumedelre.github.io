---
title: "API Platform 4.1: strict query params, multi-spec OpenAPI, and GraphQL limits"
date: 2025-02-28
series: ["api-platform-releases"]
part: 7
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.1 makes strict query parameter validation the default, introduces x-apiplatform-tags for splitting one API into multiple OpenAPI specs, and adds depth and complexity limits for GraphQL."
---

API Platform 4.1 arrived in February 2025 with a batch of features that are less about new capabilities and more about making the existing ones production-ready. Strict query params become the default. OpenAPI gains a mechanism for splitting large APIs into separate specs. GraphQL gets the abuse prevention controls it was missing.

## Strict query parameter validation is now the default

3.3 introduced query parameter validation as opt-in. 3.4 deprecated the loose behavior. 4.1 flips the switch: unknown query parameters now return 400 by default.

If your API has operations that intentionally accept undeclared parameters — analytics tracking, feature flags, pass-through proxies — you need to declare them explicitly:

```php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    parameters: [
        new QueryParameter(key: 'utm_source', required: false, schema: ['type' => 'string']),
        new QueryParameter(key: 'feature_flag', required: false, schema: ['type' => 'string']),
    ]
)]
class Book {}
```

Declared parameters pass through; undeclared parameters are rejected. The behavior can be disabled per-operation with `extraProperties: ['unknown_query_parameters' => true]` if you have a legitimate reason to accept arbitrary parameters.

## `x-apiplatform-tags` for multi-spec OpenAPI

Large APIs often need multiple OpenAPI specs: one per team, one per API version, one internal and one public. Before 4.1, the generated spec was one document, and splitting it required post-processing or separate API Platform instances.

4.1 adds an `x-apiplatform-tags` vendor extension. You tag operations with logical group names, then request the spec filtered to one or more groups:

```php
use ApiPlatform\Metadata\GetCollection;

#[GetCollection(
    extraProperties: ['x-apiplatform-tags' => ['public', 'v2']]
)]
class Book {}
```

Requesting `/api/docs.json?tags[]=public` returns only the operations tagged `public`. The full spec is still available without a filter. Groups do not affect the actual API behavior — they are a documentation-layer concern only.

This makes it feasible to maintain one API Platform configuration while serving different spec views to different consumers: a public Swagger UI, a partner portal, and an internal tool that exposes admin endpoints.

## HTTP authentication in Swagger UI

Before 4.1, the Swagger UI bundled with API Platform supported Bearer token authentication via its "Authorize" dialog. API Key and HTTP Basic authentication were not wired in.

4.1 adds configuration for multiple security schemes:

```yaml
api_platform:
    openapi:
        security_schemes:
            basicAuth:
                type: http
                scheme: basic
            apiKey:
                type: apiKey
                in: header
                name: X-API-KEY
```

Each declared scheme appears in Swagger UI's "Authorize" dialog and is applied to requests made from the UI. This is a documentation and developer experience improvement — the actual authentication logic in your application is not affected.

## GraphQL query depth and complexity limits

GraphQL's recursive query structure makes it trivial to craft a query that is small in bytes but enormous in execution cost. Without limits, a nested query four levels deep across a many-to-many relation can hit the database hundreds of times.

4.1 adds configurable depth and complexity limits:

```yaml
api_platform:
    graphql:
        max_query_depth: 10
        max_query_complexity: 100
```

`max_query_depth` is the maximum nesting level. `max_query_complexity` assigns a cost to each field and rejects queries whose total cost exceeds the threshold. Queries that exceed either limit are rejected before execution with a 400 response.

There is no universally correct value for these limits — they depend on your schema shape and expected query patterns. The defaults are intentionally permissive to avoid breaking existing APIs on upgrade. Tightening them is a deliberate configuration choice.

## Operation-level output formats

4.0 and earlier configured accepted and returned content types at the API level. 4.1 lets you narrow this per operation:

```php
use ApiPlatform\Metadata\GetCollection;

#[GetCollection(
    outputFormats: ['jsonld' => ['application/ld+json']],
    inputFormats: ['json' => ['application/json']],
)]
class Book {}
```

Operations that do not specify formats inherit the API-level configuration. This is useful for endpoints that need to return a specific format (a CSV export, a binary stream) without changing the defaults for the rest of the API.
