---
title: "API Platform 3.3: headers, link security, and OpenAPI webhooks"
date: 2024-04-29
series: ["api-platform-releases"]
part: 4
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.3 adds declarative header configuration, fine-grained link security on sub-resources, and OpenAPI webhook support."
---

API Platform 3.3 shipped in April 2024 with a set of targeted additions. None of them reshape the architecture — 3.2 already closed that chapter. What 3.3 adds is control over things that were previously either hardcoded or required a workaround: response headers, link visibility on sub-resources, and webhooks in the generated spec.

## Declarative header configuration

Before 3.3, setting custom response headers required either a custom processor that modified the response object or a Symfony event listener on `kernel.response`. Both approaches worked but lived outside the resource definition.

3.3 adds a `headers` parameter to operation metadata:

```php
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\HeaderParameter;

#[Get(
    headers: [
        new HeaderParameter(name: 'X-Custom-Header', description: 'A custom header'),
    ]
)]
```

For headers that vary per response (like `Cache-Control` with a computed max-age), the processor can still set them directly on the response. The `headers` parameter is primarily for documenting expected headers in the OpenAPI spec and for static header values.

## Link security on sub-resources

When a resource exposes links to related resources, those links appear in the serialized output regardless of whether the current user can access the linked resource. This creates a disclosure problem: a user who can read a book but not its author profile still sees the author's URI in the response.

3.3 adds security expressions to the `Link` descriptor:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\Link;

#[ApiResource]
#[Get]
class Book
{
    #[Link(
        toClass: Author::class,
        security: "is_granted('ROLE_ADMIN')"
    )]
    public Author $author;
}
```

The link is omitted from the response when the security expression evaluates to false. The linked resource itself is not affected — only whether the current response includes the reference to it.

## `ApiProperty::security`

The same security expression mechanism is available at the property level via `ApiProperty::security`. This lets you hide individual fields based on the current user without writing a custom normalizer:

```php
use ApiPlatform\Metadata\ApiProperty;

class Book
{
    #[ApiProperty(security: "is_granted('ROLE_ADMIN')")]
    public string $internalNote;
}
```

The property is excluded from serialization when the expression is false. This is cleaner than a normalizer for the common case of role-gated fields.

## OpenAPI webhooks

[OpenAPI 3.1](https://spec.openapis.org/oas/v3.1.0) supports webhooks — outbound HTTP calls that your API makes to registered listeners — in the spec document itself. Before 3.3, there was no way to document these in API Platform's generated spec.

3.3 adds a `webhooks` configuration option where you can declare the shape of your outbound calls:

```yaml
# config/packages/api_platform.yaml
api_platform:
    openapi:
        webhooks:
            bookCreated:
                post:
                    requestBody:
                        content:
                            application/json:
                                schema:
                                    $ref: '#/components/schemas/Book'
```

The webhook definitions appear in the generated spec alongside regular paths. Swagger UI renders them in a separate section.

## Swagger UI deep linking

Swagger UI supports deep linking — bookmarkable URLs that open directly to a specific operation in the interface. Before 3.3, the API Platform integration did not enable this. 3.3 turns on the Swagger UI `deepLinking` option:

```yaml
api_platform:
    swagger_ui:
        deep_linking: true
```

With this enabled, the URL fragment updates as you navigate the UI, and pasting or sharing the URL opens the same operation. Useful when writing docs that link directly to a specific endpoint.

## Strict query parameter validation

3.3 tightens the query parameter validator: parameters not declared on the operation now return a 400 response instead of being silently ignored. This behavior is opt-in:

```yaml
api_platform:
    validator:
        query_parameter_validation: true
```

The intent is to catch typos and API misuse early. If you rely on pass-through query parameters for custom logic (logging, feature flags), you need to declare them explicitly on the operation before enabling this.
