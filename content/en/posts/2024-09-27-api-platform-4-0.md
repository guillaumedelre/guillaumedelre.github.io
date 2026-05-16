---
title: "API Platform 4.0: Laravel support and PUT rethought"
date: 2024-09-27
series: ["api-platform-releases"]
part: 6
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.0 brings first-class Laravel support with Eloquent and policies, and removes PUT from default operations to fix a long-standing semantic ambiguity."
---

API Platform 4.0 shipped nine days after 3.4, in late September 2024. The version number is honest: there is no new architecture, and the migration from 3.4 is short if you resolved the deprecations. What makes this a major is the scope change — API Platform is no longer a Symfony-only framework — and one opinionated default that reverses six years of PUT behavior.

## Laravel as a first-class target

Since its first release, API Platform was built on Symfony. The HTTP layer, metadata, serializer, and Doctrine bridge all assumed Symfony's container, event dispatcher, and request lifecycle. Laravel users could run API Platform through a thin adapter, but filters, security, and Doctrine integration did not work on Eloquent.

4.0 ships a dedicated Laravel bridge. It maps API Platform's state layer onto Laravel's request lifecycle, integrates with Eloquent models directly, and wires into Laravel's authorization system:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use Illuminate\Database\Eloquent\Model;

#[ApiResource(
    operations: [new GetCollection(), new Get()]
)]
class Book extends Model
{
    protected $fillable = ['title', 'author'];
}
```

Authorization uses Laravel policies and gates rather than Symfony's security voters:

```php
#[Get(security: "policy('view', object)")]
class Book extends Model {}
```

The `policy()` expression calls Laravel's `Gate::check()` with the model instance. The Doctrine Orm filters — `SearchFilter`, `RangeFilter`, `OrderFilter` — are ported to Eloquent. Pagination, sorting, and validation work through Laravel's native mechanisms.

This is not a compatibility shim. The Laravel bridge is maintained alongside the Symfony bridge and is covered by the same test suite. Projects using either framework get the same resource definition API.

## PUT removed from default operations

Since API Platform 1.0, `#[ApiResource]` without an explicit `operations` array generated CRUD operations including PUT. The PUT handler updated existing resources and, after 3.1, could also create them via `allowCreate: true`.

4.0 removes PUT from the default set. `#[ApiResource]` now generates GET, POST, PATCH, and DELETE. To use PUT, you must declare it explicitly:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Put;

#[ApiResource(
    operations: [
        // ... other operations
        new Put(),
    ]
)]
class Book {}
```

The motivation is semantic clarity. PATCH replaces PUT for most partial-update use cases. PUT's semantics — replace the entire resource representation — are rarely what an API actually implements, but the default made it appear in every API unless actively removed. Making PUT opt-in aligns the defaults with how HTTP semantics are actually used in practice.

## PHP 8.2 minimum

4.0 drops PHP 8.0 and 8.1. PHP 8.2 is the new minimum. The readonly class syntax, `AllowDynamicProperties`, and DNF[^dnf] types introduced in 8.2 are available throughout the codebase. No specific 8.2 feature is load-bearing for 4.0 — the version bump is primarily about dropping the older maintenance burden.

## Symfony 7 and Doctrine ORM 3 minimum

On the Symfony side, 4.0 requires Symfony 7.0 and Doctrine ORM 3. Both were already supported in 3.4. The migration from 3.4 to 4.0 on the Symfony track is: resolve 3.4 deprecations, verify you are on Symfony 7 and ORM 3, then upgrade. No new migration work is required if those are already in place.

## What 4.0 is not

4.0 is not a new architecture. The state providers, processors, and resource metadata model from 3.0 are unchanged. The Laravel bridge adds a new execution context but does not change how resources or operations are declared. The split is intentional: if 3.0 was the "what", 4.0 is the "where".

[^dnf]: Disjunctive Normal Form types: intersection types combined with union, like `(A&B)|null`.
