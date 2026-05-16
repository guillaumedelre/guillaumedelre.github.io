---
title: "API Platform 3.1: your resource doesn't have to be your entity"
date: 2023-01-23
series: ["api-platform-releases"]
part: 2
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.1 decouples API resources from Doctrine entities, ships a spec-compliant PUT, and collects denormalization errors as a list."
---

Four months after 3.0, API Platform 3.1 arrived with the first batch of features built on the new state model. Not every change is dramatic, but one of them solves a problem that drove a lot of convoluted workarounds in 2.x: your API resource no longer needs to be your Doctrine entity.

## The resource/entity split

In 2.x, API Platform worked best when your API resource and your persistence model were the same class. Using a DTO as the API surface was possible through the Input/Output DTO system, but that system was removed in 3.0 — it complicated the state model without enough benefit.

3.1 replaces it with something cleaner. The `stateOptions` parameter on an operation accepts a `DoctrineOrmOptions` object that points to a different entity:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Doctrine\Orm\State\Options;

#[ApiResource(
    operations: [
        new GetCollection(
            stateOptions: new Options(entityClass: BookEntity::class),
        ),
    ]
)]
class BookDto
{
    public string $title;
    public string $author;
}
```

The provider receives the `BookEntity` objects from Doctrine and the serialization layer maps them to `BookDto`. The Doctrine filters, pagination, and ordering all work on `BookEntity`. The API surface exposes `BookDto`. The two can evolve independently.

This matters more than it looks. Your persistence model accumulates internal fields, relations, and columns that have no business being in your API. Before 3.1, you either exposed them anyway or built an elaborate normalizer to hide them. Now you declare what the API looks like as a separate class and let the framework handle the mapping.

## PUT that follows the spec

Since version 1.0, API Platform's PUT handler updated existing resources. Creating a resource via PUT — which the HTTP spec explicitly allows — was not supported. 3.1 adds `uriTemplate`-based creation:

```php
#[Put(
    uriTemplate: '/books/{id}',
    allowCreate: true,
)]
```

With `allowCreate: true`, a PUT to a URI that does not exist creates the resource instead of returning 404. The identifier comes from the URI, not from the request body. This is what RFC 9110 describes for PUT: "If the target resource does not have a current representation and the PUT successfully creates one, then the origin server MUST inform the user agent by sending a 201 (Created) response."

It is a small flag, but it opens API Platform to use cases — idempotent resource creation, client-assigned identifiers — that previously required a custom controller.

## Denormalization errors collected, not thrown

Before 3.1, deserialization errors stopped at the first problem. Send a request body with five invalid fields and get an error about the first one. Fix it, send again, find the second. Repeat five times.

3.1 adds a `collect_denormalization_errors` option on the operation that changes this behavior:

```php
#[Post(collectDenormalizationErrors: true)]
```

With this enabled, API Platform catches all type errors and constraint violations during deserialization and returns them as a structured list in the response, formatted the same way as validation errors. One round-trip, full picture.

## `ApiResource::openapi` replaces `openapiContext`

The old `openapiContext` parameter accepted a raw array that was merged into the generated OpenAPI schema — convenient but untyped. 3.1 introduces a first-class `openapi` parameter that accepts an `OpenApiOperation` object:

```php
use ApiPlatform\OpenApi\Model\Operation;
use ApiPlatform\OpenApi\Model\RequestBody;

#[Post(
    openapi: new Operation(
        requestBody: new RequestBody(
            description: 'Create a book',
            required: true,
        ),
        summary: 'Create a new book entry',
    )
)]
```

The old `openapiContext` array still works but is deprecated. The new approach is typed, IDE-friendly, and validates at construction time rather than at schema generation time. PHP 8.1 backed enums also get proper OpenAPI schema generation in 3.1 — a field typed as a backed enum produces a schema with `enum` values and the correct type, without any annotation.

## The pattern is clear

3.0 established the architecture. 3.1 shows what that architecture enables: clean resource/entity separation without a parallel DTO system, RFC-correct HTTP semantics, better error reporting. None of these would have been as clean to implement on the 2.x data provider model. The features in 3.1 are the first proof that the rewrite was the right call.
