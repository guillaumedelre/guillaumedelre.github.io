---
title: "API Platform 3.0: a new state model and the end of DataProviders"
date: 2022-11-18
series: ["api-platform-releases"]
part: 1
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.0 replaced DataProviders and DataPersisters with a state model that makes HTTP operations explicit — and required PHP 8.1 and Symfony 6 to do it."
---

API Platform 3.0 arrived in November 2022 with Symfony 6.1 as a hard minimum and a core architecture that looked nothing like 2.x. The migration guide is long. The reason it's long is interesting.

The old model had a conceptual leak. `DataProviderInterface` and `DataPersisterInterface` were called for every HTTP request, but the provider received the operation context as a hint — not as a contract. A collection provider and an item provider were separate interfaces, but both lived in the same mental bucket: "things that return data." The HTTP layer knew what was being requested; the provider had to reconstruct that knowledge from context clues passed in the `$context` array.

3.0 inverts the model. Operations are declared first. Data access is wired to operations.

## State providers replaced data providers

The old `DataProviderInterface` is gone. The replacement is `ProviderInterface`:

```php
use ApiPlatform\State\ProviderInterface;
use ApiPlatform\Metadata\Operation;

class BookProvider implements ProviderInterface
{
    public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
    {
        if ($operation instanceof CollectionOperationInterface) {
            return $this->repository->findAll();
        }

        return $this->repository->find($uriVariables['id']);
    }
}
```

The difference is not syntactic. In 2.x, you registered a provider and API Platform called it for any matching resource. In 3.0, you bind a provider to a specific operation. The provider no longer guesses what triggered it — the operation object it receives is the contract.

## State processors replaced data persisters

`DataPersisterInterface` had the same problem on the write side: one class handling create, update, and delete, distinguishing them by inspecting the HTTP method or the object state. `ProcessorInterface` receives the operation as a typed argument:

```php
use ApiPlatform\State\ProcessorInterface;
use ApiPlatform\Metadata\Operation;

class BookProcessor implements ProcessorInterface
{
    public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = []): mixed
    {
        $this->entityManager->persist($data);
        $this->entityManager->flush();

        return $data;
    }
}
```

More usefully: you can bind a different processor per operation. The delete operation gets one that removes. The post operation gets one that validates and stores. No switch statement, no method inspection, no shared class trying to be three things at once.

## Operations declared explicitly in PHP 8.1 attributes

The other half of 3.0 is the metadata layer. Doctrine annotations are replaced by PHP 8.1 native attributes, and each operation is declared explicitly on the resource class:

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\Post;

#[ApiResource(
    operations: [
        new GetCollection(provider: BookProvider::class),
        new Get(provider: BookProvider::class),
        new Post(processor: BookProcessor::class),
    ]
)]
class Book
{
    // ...
}
```

This is more verbose than `@ApiResource` with magic defaults. It is also explicit. You know exactly what HTTP operations exist for this resource, what retrieves data, what writes it, and where the logic lives. The defaults of 2.x were convenient until the day you needed to override one and couldn't figure out which service to decorate without reading the source.

## PHP 8.1 was not a coincidence

The hard requirement for PHP 8.1 is load-bearing. Readonly properties let API Platform enforce that operation metadata is immutable once constructed. Fibers underpin the async handling in the Mercure integration. Enums clarify what properties like `paginationType` actually accept. First-class callables make filter registration cleaner.

More practically: the full expression of 3.0's architecture — readonly state, typed operations, operation-scoped providers — needed 8.1 to not feel like workarounds. Dropping PHP 7.x and 8.0 was not a housekeeping decision.

## The migration is real work

The jump from 2.x to 3.0 is not a version bump. Every `DataProvider` becomes a `ProviderInterface`. Every `DataPersister` becomes a `ProcessorInterface`. Annotations become attributes. Custom normalizers and filters may need restructuring. The upgrade guide documents all of it, but "documented" does not mean "fast."

What you get on the other side is an architecture that scales without the ambient complexity of 2.x: no more guessing which interface to implement, no more `$this->supports()` chains, no more invisible defaults quietly overriding explicit config.

3.0 is the API Platform you'd design from scratch knowing what you know after years of 2.x. The price is the migration. The version number is honest about that.
