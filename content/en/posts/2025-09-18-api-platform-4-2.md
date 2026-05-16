---
title: "API Platform 4.2: JSON streamer, ObjectMapper, and autoconfigure"
date: 2025-09-18
series: ["api-platform-releases"]
part: 8
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.2 streams large JSON collections without buffering, replaces manual DTO mapping with ObjectMapper, and autoconfigures resources from their class attributes."
---

API Platform 4.2 arrived in September 2025. Three changes stand out: a JSON streamer for large collections that avoids buffering the entire response in memory, an ObjectMapper that replaces the manual wiring in `stateOptions`-based DTO flows, and autoconfiguration of `#[ApiResource]` without explicit service registration.

## JSON streamer for large collections

The default Symfony serializer builds the full response in memory before writing it to the output. For a collection of 10,000 items, this means allocating a PHP array, serializing it to a string, and keeping both in memory until the response is flushed. At scale, this is the source of the OOM errors that force people to add pagination everywhere.

4.2 adds a streaming JSON encoder that writes the response incrementally:

```yaml
api_platform:
    serializer:
        enable_json_streamer: true
```

With streaming enabled, the response is written directly to the output buffer as each item is serialized. Memory usage stays roughly constant regardless of collection size. The trade-off: you cannot set response headers after streaming starts, and the HTTP status code must be committed before the first byte is written.

## ObjectMapper replaces manual DTO wiring

3.1 introduced `stateOptions` with `DoctrineOrmOptions` for separating the API resource from the Doctrine entity. The provider received entity objects and the serializer mapped them to the DTO. This worked, but the mapping was implicit — the serializer used property names to match fields, and anything that did not match was either ignored or caused a normalization error.

4.2 introduces `ObjectMapper`, a declarative mapping layer between entities and DTOs:

```php
use Symfony\Component\ObjectMapper\Attribute\Map;

#[Map(source: BookEntity::class)]
class BookDto
{
    public string $title;
    public string $authorName;
}
```

The `#[Map]` attribute tells ObjectMapper that `BookDto` can be populated from `BookEntity`. Field names are matched by convention; mismatches are declared explicitly at the property level:

```php
use Symfony\Component\ObjectMapper\Attribute\Map;

#[Map(source: BookEntity::class)]
class BookDto
{
    #[Map(source: 'author.fullName')]
    public string $authorName;
}
```

The dot notation traverses nested objects. The mapping runs before serialization and replaces the implicit property-matching behavior of the serializer. Unmapped fields raise an error at configuration time, not at runtime.

ObjectMapper works with `stateOptions`:

```php
use ApiPlatform\Doctrine\Orm\State\Options;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;
use Symfony\Component\ObjectMapper\Attribute\Map;

#[ApiResource(
    operations: [
        new GetCollection(
            stateOptions: new Options(entityClass: BookEntity::class),
        ),
    ]
)]
#[Map(source: BookEntity::class)]
class BookDto {}
```

The provider fetches `BookEntity` objects from Doctrine. ObjectMapper converts them to `BookDto` instances. The serializer writes the DTO. Three distinct steps, each with a clear contract.

## TypeInfo integration throughout the stack

Symfony 7.1 introduced the [TypeInfo component](https://symfony.com/doc/current/components/type_info.html), a unified type introspection layer that understands union types, intersection types, generic collections, and nullable types across reflection, PHPDoc, and PHP 8.x syntax.

4.2 replaces API Platform's internal type resolution with TypeInfo. This affects filter schema generation, OpenAPI schema inference, and the serializer's type coercion. The visible benefit is that types that previously generated incorrect or missing OpenAPI schemas — `Collection<int, Book>`, `list<string>`, intersection types — now produce accurate schemas without manual `@ApiProperty` annotations.

## Autoconfigure `#[ApiResource]`

Before 4.2, adding `#[ApiResource]` to a class was sufficient for Hugo to discover it only if the class was in a path scanned by API Platform's resource loader. Outside that path, you needed explicit service configuration.

4.2 hooks into Symfony's autoconfigure system. Any class tagged with `#[ApiResource]` is automatically registered as a resource regardless of its location, as long as it is in a directory covered by Symfony's component scan. No `config/services.yaml` entry needed.

For Laravel, the equivalent uses Laravel's service provider autoloading — Eloquent models with `#[ApiResource]` are picked up automatically without manual registration.

## Doctrine ExistsFilter

The `ExistsFilter` constrains a collection by whether a nullable relation or field is set:

```php
#[ApiFilter(ExistsFilter::class, properties: ['publishedAt'])]
class Book {}
```

`GET /books?exists[publishedAt]=true` returns books where `publishedAt` is not null. `exists[publishedAt]=false` returns books where it is null.
