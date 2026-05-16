---
title: "API Platform 3.4 : BackedEnum comme ressources et support DBAL 4"
date: 2024-09-18
series: ["api-platform-releases"]
part: 5
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.4 fait des BackedEnum des ressources API complètes, ajoute un BackedEnumFilter, supporte les expressions de sécurité sur les paramètres, et ajoute le support DBAL 4."
---

API Platform 3.4 est arrivé en septembre 2024 comme dernier mineur avant le saut vers la 4.0. La fonctionnalité principale est le BackedEnum comme ressource complète — pas juste un champ typé, mais un enum qui est lui-même un endpoint API.

## BackedEnum comme ressources API

Depuis PHP 8.1, les classes BackedEnum ont un ensemble fixe de cas avec des valeurs de support string ou integer. API Platform 3.4 permet de mettre `#[ApiResource]` directement sur un BackedEnum :

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

Un endpoint `GET /book_statuses` retourne la liste des cas. Chaque cas est sérialisé avec son nom et sa valeur. L'endpoint est en lecture seule — les enums sont immuables par nature.

C'est surtout utile pour les consommateurs frontend qui veulent une liste lisible par machine des valeurs valides sans les coder en dur. L'alternative était un contrôleur personnalisé ou une ressource DTO dédiée listant manuellement les valeurs de l'enum.

## BackedEnumFilter

Le compagnon des ressources enum est `BackedEnumFilter`, un nouveau filtre pour les collections Doctrine qui contraint une requête par une propriété BackedEnum :

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

`GET /books?status=published` filtre la collection aux livres où `status` est égal à `BookStatus::Published`. Les valeurs d'enum invalides retournent une réponse 400. Avant ce filtre, on devait soit écrire un filtre personnalisé, soit utiliser `SearchFilter` et valider la valeur manuellement.

## Expressions de sécurité sur les paramètres

La 3.3 avait ajouté la sécurité aux liens et aux propriétés. La 3.4 étend cela aux paramètres de requête. Un paramètre peut déclarer une expression de sécurité qui contrôle s'il est accepté :

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

Quand l'expression de sécurité est false, le paramètre est rejeté avec un 403, pas silencieusement ignoré. C'est plus explicite que vérifier le rôle de l'utilisateur dans le provider après avoir reçu le paramètre.

## Support DBAL 4 ajouté

La 3.4 ajoute le support de Doctrine DBAL 4, qui apporte des changements au système de types qui affectent la façon dont les types personnalisés et le SQL spécifique à la plateforme fonctionnent. Les filtres Doctrine Orm et les extensions de requête dans API Platform ont été mis à jour pour fonctionner avec la nouvelle API de types DBAL 4.

DBAL 3 (`^3.4.0`) et DBAL 4 sont supportés simultanément en 3.4. C'est la release à adopter pour migrer vers DBAL 4 tout en restant sur une branche stable API Platform 3.x.

## Validateur de paramètres de requête déprécié

La 3.3 avait ajouté le validateur strict de paramètres de requête en opt-in. La 3.4 déprécie l'ancien comportement (paramètres inconnus silencieusement ignorés) en préparation pour rendre la validation stricte la valeur par défaut en 4.0. Les projets qui s'appuient sur des paramètres de requête pass-through ont une release de plus pour les déclarer explicitement.

## Dernier arrêt avant la 4.0

La 3.4 est la dernière release 3.x avec de nouvelles fonctionnalités. Tout ce qui était déprécié d'ici la 3.4 disparaît en 4.0. Le chemin de migration de la 3.4 vers la 4.0 est intentionnellement court : résoudre les dépréciations, puis mettre à jour.
