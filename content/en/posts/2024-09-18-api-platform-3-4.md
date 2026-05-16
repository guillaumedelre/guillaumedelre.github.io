---
title: "API Platform 3.4: BackedEnum as resources and DBAL 4 support"
date: 2024-09-18
series: ["api-platform-releases"]
part: 5
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.4 makes BackedEnum classes full API resources, adds a BackedEnumFilter, supports security expressions on parameters, and adds DBAL 4 support."
---

API Platform 3.4 landed in September 2024 as the last minor before the 4.0 jump. The headline feature is BackedEnum as full resources — not just a typed field, but an enum that is itself an API endpoint.

## BackedEnum as API resources

Since PHP 8.1, BackedEnum classes have a fixed set of cases with string or integer backing values. API Platform 3.4 lets you put `#[ApiResource]` directly on a BackedEnum:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [new GetCollection()]
)]
enum BookStatus: string
{
    case Draft = 'draft';
    case Published = 'published';
    case Archived = 'archived';
}
```

A `GET /book_statuses` endpoint returns the list of cases. Each case is serialized with its name and value. The endpoint is read-only — enums are immutable by nature.

This is mostly useful for frontend consumers that want a machine-readable list of valid values without hardcoding them. The alternative was a custom controller or a dedicated DTO resource listing the enum values manually.

## BackedEnumFilter

The companion to enum resources is `BackedEnumFilter`, a new filter for Doctrine collections that constrains a query by a BackedEnum property:

```php
use ApiPlatform\Doctrine\Orm\Filter\BackedEnumFilter;
use ApiPlatform\Metadata\ApiFilter;
use ApiPlatform\Metadata\ApiResource;

#[ApiResource]
#[ApiFilter(BackedEnumFilter::class, properties: ['status'])]
class Book
{
    public BookStatus $status;
}
```

`GET /books?status=published` filters the collection to books where `status` equals `BookStatus::Published`. Invalid enum values return a 400 response. Before this filter, you had to either write a custom filter or use `SearchFilter` and validate the value manually.

## Security expressions on parameters

3.3 added security to links and properties. 3.4 extends this to query parameters. A parameter can declare a security expression that controls whether it is accepted at all:

```php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    parameters: [
        new QueryParameter(
            key: 'includeDeleted',
            security: "is_granted('ROLE_ADMIN')"
        ),
    ]
)]
class Book {}
```

When the security expression is false, the parameter is rejected with a 403, not silently ignored. This is more explicit than checking the user's role inside the provider after receiving the parameter.

## DBAL 4 support added

3.4 adds support for Doctrine DBAL 4, which ships with type system changes that affect how custom types and platform-specific SQL work. The Doctrine Orm filters and query extensions in API Platform were updated to work with the new DBAL 4 type API.

Both DBAL 3 (`^3.4.0`) and DBAL 4 are supported simultaneously in 3.4. This is the release to upgrade to if you want to adopt DBAL 4 while staying on a stable API Platform 3.x branch.

## Query parameter validator deprecated

3.3 added the strict query parameter validator as an opt-in. 3.4 deprecates the old behavior (unknown parameters silently ignored) in preparation for making strict validation the default in 4.0. Projects that relied on pass-through query parameters have one more release to declare them explicitly.

## Last stop before 4.0

3.4 is the last 3.x release with new features. Anything 3.x that was deprecated by 3.4 is gone in 4.0. The migration path from 3.4 to 4.0 is intentionally short: resolve the deprecations, then upgrade.
