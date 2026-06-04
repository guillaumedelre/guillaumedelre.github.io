---
title: "API Platform 4.3: MCP server, Scalar UI, and security before the provider"
date: 2026-03-23
series: ["api-platform-releases"]
part: 9
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.3 exposes your resources to AI agents via MCP, ships Scalar as an alternative API reference UI, and evaluates security expressions before hitting the database."
translationKey: "api-platform-4-3"
---

API Platform 4.3 shipped March 13, 2026. The headline addition is MCP support: your API resources can now be exposed as tools and resources for LLM agents with no extra infrastructure. Alongside that, two quality-of-life improvements stand out — Scalar as an alternative documentation UI, and security checks that fire before the state provider runs.

## MCP server support

The Model Context Protocol is the emerging standard for connecting LLMs to external tools and data sources. 4.3 ships an experimental MCP integration that maps directly onto the API Platform resource model.

Two new attributes in `ApiPlatform\Metadata` handle the exposure:

**`McpTool`** marks an operation as an MCP tool that an agent can call:

```php
use ApiPlatform\Metadata\McpTool;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource]
#[McpTool(name: 'list_books', description: 'Returns a list of available books')]
#[GetCollection]
class Book {}
```

**`McpResource`** exposes a resource as addressable MCP content, identified by a URI:

```php
use ApiPlatform\Metadata\McpResource;

#[ApiResource]
#[McpResource(uri: 'books://{id}', mimeType: 'application/ld+json')]
class Book {}
```

Both attributes repeat and compose with existing API Platform operations. The MCP server format is configurable:

```yaml
api_platform:
    mcp:
        format: 'jsonld'
```

The integration lives in a separate `api-platform/mcp` package. Collections are supported. Both Symfony and Laravel are covered.

## Scalar as an API reference UI

Swagger UI has been the default documentation interface since API Platform 2.x. 4.3 adds Scalar as an opt-in alternative. Scalar renders OpenAPI specs with a more modern interface and better readability for large APIs.

Enable it in your config (requires TwigBundle):

```yaml
api_platform:
    enable_scalar: true
    openapi:
        scalar_extra_configuration:
            theme: 'purple'
            darkMode: true
```

## Security before the provider

Before 4.3, the `security:` expression on an operation was evaluated after the state provider had already fetched the object from the database. An unauthorized request still triggered a Doctrine query before being rejected. At scale, this mattered.

4.3 registers a new `AccessCheckerProvider` decorator that runs at priority 10, before the read step:

```php
#[ApiResource]
#[Get(security: "is_granted('ROLE_ADMIN')")]
class Book {}
```

The `is_granted` check now fires before any database query. The trade-off: `$object` is `null` at the `pre_read` stage. Expressions that depend on the loaded object — `security: "object.getOwner() == user"` — need to move to `securityPostDenormalize`, which still runs after the provider.

This is a behavioral change. Review your security expressions if they reference `object`.

## ComparisonFilter

A new `ComparisonFilter` decorator in `ApiPlatform\Doctrine\Orm\Filter` wraps any equality filter and adds comparison operators (`gt`, `gte`, `lt`, `lte`, `ne`):

```php
use ApiPlatform\Doctrine\Orm\Filter\ComparisonFilter;
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(filter: new ComparisonFilter(new RangeFilter()), property: 'price')]
class Product {}
```

This generates OpenAPI parameters for each operator variant automatically. The filter is marked experimental.

## UuidFilter and nested relation support

A new `UuidFilter` in `ApiPlatform\Doctrine\Orm\Filter` filters collections by UUID properties, including nested relations. The `IriFilter` receives the same nested relation support in this release.

```php
use ApiPlatform\Doctrine\Orm\Filter\UuidFilter;
use ApiPlatform\Metadata\ApiFilter;

#[ApiResource]
#[ApiFilter(UuidFilter::class, properties: ['author.uuid'])]
class Book {}
```

## PartialSearchFilter with case sensitivity control

The new `PartialSearchFilter` in `ApiPlatform\Doctrine\Orm\Filter` performs partial string matching. By default it wraps the comparison in `LOWER()` for case-insensitive results. Pass `caseSensitive: true` to disable this:

```php
use ApiPlatform\Doctrine\Orm\Filter\PartialSearchFilter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(filter: new PartialSearchFilter(caseSensitive: true), property: 'title')]
class Book {}
```

## ODM SortFilter with nested property support

A `SortFilter` arrives for MongoDB (ODM) in `ApiPlatform\Doctrine\Odm\Filter`, designed exclusively for use with the `Parameters` system. It supports nested properties out of the box:

```php
use ApiPlatform\Doctrine\Odm\Filter\SortFilter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(filter: new SortFilter(), property: 'department.name')]
class Employee {}
```

## Readonly Doctrine entities drop PUT and PATCH automatically

If a Doctrine entity is declared read-only with `#[ORM\Entity(readOnly: true)]`, API Platform now automatically removes the `PUT` and `PATCH` operations from its resource definition. No manual operation list needed:

```php
use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource]
#[ORM\Entity(readOnly: true)]
class AuditLog {}
```

`GET`, `GetCollection`, and `DELETE` remain available. Only write operations are stripped.

## Maker namespace prefix

The Symfony bundle accepts a new config key that prefixes all classes generated by API Platform makers:

```yaml
api_platform:
    maker:
        namespace_prefix: 'Api'
```

With this set, `make:api-resource` generates classes under `App\Api\` instead of `App\`.

## QueryParameter default values

`QueryParameter` now accepts a `default` option that provides a fallback value when the parameter is absent from the request:

```php
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[GetCollection]
#[QueryParameter(key: 'order', default: 'asc')]
class Book {}
```

## UUID and ULID validation on query parameters

Query parameters and headers declared with a `uuid` or `ulid` format are now automatically validated. Malformed identifiers are rejected before they reach the state provider, without any extra constraint needed.

## ProblemExceptionInterface

Custom exceptions can now implement `ProblemExceptionInterface` to map directly to the [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) Problem Details fields (`type`, `title`, `detail`, and extension members). Previously, this required a custom exception normalizer.

## LDP headers

API Platform now automatically adds `Accept-Post` and `Allow` response headers on collection and item endpoints, following the [Linked Data Platform specification](https://www.w3.org/TR/ldp/). This improves discoverability for hypermedia clients that rely on these headers to determine which operations are available on a given resource.

## Breaking changes

A few behavioral changes to be aware of when upgrading:

- **Security runs before the provider.** Expressions that reference `object` will receive `null`. Move them to `securityPostDenormalize`.
- **Doctrine filters require an explicit `property`.** A missing `property` attribute now throws an exception instead of failing silently.
- **JSON-LD `@type` with `output` and `itemUriTemplate`** uses a different class name to be semantically consistent.
