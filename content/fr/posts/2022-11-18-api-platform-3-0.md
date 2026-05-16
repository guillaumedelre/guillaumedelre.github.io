---
title: "API Platform 3.0 : un nouveau modèle d'état et la fin des DataProviders"
date: 2022-11-18
series: ["api-platform-releases"]
part: 1
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.0 a remplacé les DataProviders et DataPersisters par un modèle d'état qui rend les opérations HTTP explicites — et a exigé PHP 8.1 et Symfony 6 pour y arriver."
---

API Platform 3.0 est arrivé en novembre 2022 avec Symfony 6.1 comme prérequis strict et une architecture de base qui ne ressemblait en rien à la 2.x. Le guide de migration est long. La raison pour laquelle il est long est intéressante.

L'ancien modèle avait une fuite conceptuelle. `DataProviderInterface` et `DataPersisterInterface` étaient appelés pour chaque requête HTTP, mais le provider recevait le contexte de l'opération comme un indice — pas comme un contrat. Un provider de collection et un provider d'item étaient des interfaces distinctes, mais les deux vivaient dans le même seau mental : "choses qui retournent des données." La couche HTTP savait ce qui était demandé ; le provider devait reconstruire cette connaissance à partir d'indices passés dans le tableau `$context`.

La 3.0 inverse le modèle. Les opérations sont déclarées en premier. L'accès aux données est câblé aux opérations.

## Les state providers remplacent les data providers

L'ancien `DataProviderInterface` a disparu. Le remplaçant est `ProviderInterface` :

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

La différence n'est pas syntaxique. En 2.x, on enregistrait un provider et API Platform l'appelait pour toute ressource correspondante. En 3.0, on lie un provider à une opération spécifique. Le provider n'a plus à deviner ce qui l'a déclenché — l'objet opération qu'il reçoit est le contrat.

## Les state processors remplacent les data persisters

`DataPersisterInterface` avait le même problème côté écriture : une seule classe gérant la création, la mise à jour et la suppression, les distinguant en inspectant la méthode HTTP ou l'état de l'objet. `ProcessorInterface` reçoit l'opération comme argument typé :

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

Plus utile encore : on peut lier un processor différent par opération. L'opération de suppression en reçoit un qui supprime. L'opération de création en reçoit un qui valide et stocke. Pas de switch, pas d'inspection de méthode, pas de classe partagée qui essaie d'être trois choses à la fois.

## Les opérations déclarées explicitement en attributs PHP 8.1

L'autre moitié de la 3.0 est la couche de métadonnées. Les annotations Doctrine sont remplacées par des attributs natifs PHP 8.1, et chaque opération est déclarée explicitement sur la classe ressource :

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

C'est plus verbeux que `@ApiResource` avec ses valeurs par défaut magiques. C'est aussi explicite. On sait exactement quelles opérations HTTP existent pour cette ressource, ce qui récupère les données, ce qui les écrit, et où vit la logique. Les valeurs par défaut de la 2.x étaient pratiques jusqu'au jour où il fallait en surcharger une sans réussir à déterminer quel service décorer sans lire le code source.

## PHP 8.1 n'était pas un hasard

L'exigence stricte pour PHP 8.1 est structurante. Les propriétés readonly permettent à API Platform de s'assurer que les métadonnées d'opération sont immuables une fois construites. Les fibers sous-tendent la gestion asynchrone dans l'intégration Mercure. Les enums clarifient ce que des propriétés comme `paginationType` acceptent réellement. Les callables de première classe rendent l'enregistrement des filtres plus propre.

Plus concrètement : l'expression complète de l'architecture 3.0 — état readonly, opérations typées, providers scopés à l'opération — avait besoin de la 8.1 pour ne pas ressembler à des contournements. Abandonner PHP 7.x et 8.0 n'était pas une décision de nettoyage.

## La migration est un vrai travail

Le passage de 2.x à 3.0 n'est pas un simple bump de version. Chaque `DataProvider` devient un `ProviderInterface`. Chaque `DataPersister` devient un `ProcessorInterface`. Les annotations deviennent des attributs. Les normaliseurs et filtres personnalisés peuvent nécessiter une restructuration. Le guide de mise à jour documente tout cela, mais "documenté" ne veut pas dire "rapide."

Ce qu'on obtient de l'autre côté est une architecture qui passe à l'échelle sans la complexité ambiante de la 2.x : plus besoin de deviner quelle interface implémenter, plus de chaînes `$this->supports()`, plus de valeurs par défaut invisibles qui surchargent silencieusement la config explicite.

La 3.0 est l'API Platform qu'on concevrait de zéro en sachant ce qu'on sait après des années de 2.x. Le prix est la migration. Le numéro de version est honnête là-dessus.
