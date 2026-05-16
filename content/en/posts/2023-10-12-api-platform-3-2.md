---
title: "API Platform 3.2: errors as resources and sub-resources come back"
date: 2023-10-12
series: ["api-platform-releases"]
part: 3
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.2 makes errors first-class Problem Detail resources, brings back sub-resources cleanly, and makes event listeners optional."
---

API Platform 3.2 arrived in October 2023 with three changes that pushed the state model further: errors became resources, sub-resources came back in a form that actually fits the architecture, and the last legacy extension point — event listeners — was formally replaced.

## Errors as resources

Before 3.2, error handling was outside the resource model. Exceptions were caught by a Symfony event listener and converted to a response, with limited control over the shape of the output.

3.2 makes errors first-class `ApiResource` classes compliant with [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (Problem Details for HTTP APIs). The built-in error class is `ApiPlatform\ApiResource\Error`, and you can create your own:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\ErrorResource;
use ApiPlatform\Metadata\Exception\ProblemExceptionInterface;

#[ApiResource]
#[ErrorResource]
class BookNotFoundError extends \RuntimeException implements ProblemExceptionInterface
{
    public function __construct(private readonly string $bookId)
    {
        parent::__construct("Book $bookId not found");
    }

    public function getType(): string
    {
        return '/errors/book-not-found';
    }
}
```

When this exception is thrown anywhere in the state layer, API Platform catches it, serializes it as a Problem Detail response, and generates a proper OpenAPI schema for it. The error type, title, detail, and status are all part of the resource contract — not hardcoded strings in a listener.

## Sub-resources without the workarounds

Sub-resources existed in 2.x but were removed in 3.0 because they were tightly coupled to the old data provider model and couldn't be cleanly mapped to the new operation-first architecture. 3.2 reintroduces them in a way that fits.

A sub-resource is a resource accessible through a parent resource's URI. In 3.2, it is declared directly on the child resource using `uriTemplate`:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new GetCollection(
            uriTemplate: '/books/{bookId}/reviews',
            uriVariables: [
                'bookId' => new Link(fromClass: Book::class),
            ],
        ),
    ]
)]
class Review
{
    // ...
}
```

The `Link` descriptor makes the relationship explicit. The provider receives `bookId` in `$uriVariables` and can use it to scope the query. No magic inference, no implicit joins — the URI structure and the data access are both declared.

## `canonical_uri_template` for multiple access paths

When a resource is accessible through multiple URIs (a direct endpoint and a sub-resource endpoint), OpenAPI needs to know which URI is canonical for `$ref` links. 3.2 uses the top-level `uriTemplate` on `ApiResource` as the default canonical URI. For finer control, the `canonical_uri_template` option can be passed via `extraProperties` on any operation to override it explicitly.

```php
#[ApiResource(
    uriTemplate: '/reviews/{id}',
    operations: [
        new Get(),
        new GetCollection(
            uriTemplate: '/books/{bookId}/reviews',
            uriVariables: ['bookId' => new Link(fromClass: Book::class)],
        ),
    ]
)]
class Review {}
```

The generated OpenAPI spec uses the canonical URI for schema references, keeping the document consistent when a resource appears under several paths.

## Union and intersection types

3.2 adds support for PHP union and intersection types in the metadata layer. A property declared as `Book|Magazine` generates a proper `oneOf` schema in OpenAPI. This was previously unsupported — you had to fall back to an untyped `mixed` or annotate the property manually.

## Event listeners made optional

The last compatibility shim from 2.x was the ability to use Symfony event listeners on the `kernel.request` and `kernel.view` events to intercept API Platform's data flow. 3.2 does not remove them, but introduces an opt-out: setting `event_listeners_backward_compatibility_layer: false` in the API Platform configuration disables the event-based hooks entirely. The replacement is a provider or processor decorated with another provider or processor. The event-based hook was stateful, order-dependent, and bypassed the operation context entirely. Decorated providers get the operation object and can call the inner provider when ready.

## The state model is now complete

3.0 introduced the architecture. 3.1 added resource/entity separation. 3.2 closes the remaining gaps: errors have a resource contract, sub-resources have a clean declaration model, and the state layer now covers every extension point that event listeners once handled. The 2.x shims still exist, but opting out of them is now a single config flag.
