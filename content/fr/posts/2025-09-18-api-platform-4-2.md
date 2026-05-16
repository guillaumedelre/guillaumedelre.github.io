---
title: "API Platform 4.2 : JSON streamer, ObjectMapper, et autoconfigure"
date: 2025-09-18
series: ["api-platform-releases"]
part: 8
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.2 diffuse les grandes collections JSON sans buffering, remplace le câblage DTO manuel par ObjectMapper, et autoconfigure les ressources depuis leurs attributs de classe."
---

API Platform 4.2 est arrivé en septembre 2025. Trois changements se distinguent : un JSON streamer pour les grandes collections qui évite de bufferiser toute la réponse en mémoire, un ObjectMapper qui remplace le câblage manuel dans les flux DTO basés sur `stateOptions`, et l'autoconfiguration de `#[ApiResource]` sans enregistrement de service explicite.

## JSON streamer pour les grandes collections

Le serializer Symfony par défaut construit la réponse complète en mémoire avant de l'écrire dans la sortie. Pour une collection de 10 000 éléments, cela signifie allouer un tableau PHP, le sérialiser en string, et garder les deux en mémoire jusqu'au flush de la réponse. À grande échelle, c'est la source des erreurs OOM qui forcent à ajouter de la pagination partout.

La 4.2 ajoute un encodeur JSON en streaming qui écrit la réponse de façon incrémentale :

```yaml
api_platform:
    serializer:
        use_json_streamer: true
```

Avec le streaming activé, la réponse est écrite directement dans le buffer de sortie au fur et à mesure que chaque élément est sérialisé. L'utilisation mémoire reste approximativement constante quelle que soit la taille de la collection. La contrepartie : on ne peut pas définir de headers de réponse après le début du streaming, et le code de statut HTTP doit être validé avant que le premier octet soit écrit.

Le streaming est opt-in par opération :

```php
use ApiPlatform\Metadata\GetCollection;

#[GetCollection(
    extraProperties: ['use_json_streamer' => true]
)]
class Book {}
```

## ObjectMapper remplace le câblage DTO manuel

La 3.1 avait introduit `stateOptions` avec `DoctrineOrmOptions` pour séparer la ressource API de l'entité Doctrine. Le provider recevait des objets entité et le serializer les mappait sur le DTO. Ça fonctionnait, mais le mapping était implicite — le serializer utilisait les noms de propriétés pour faire correspondre les champs, et tout ce qui ne correspondait pas était soit ignoré soit causait une erreur de normalisation.

La 4.2 introduit `ObjectMapper`, une couche de mapping déclarative entre entités et DTOs :

```php
use ApiPlatform\ObjectMapper\Map;
use ApiPlatform\ObjectMapper\ObjectMapper;

#[Map(from: BookEntity::class)]
class BookDto
{
    public string $title;
    public string $authorName;
}
```

L'attribut `#[Map]` indique à ObjectMapper que `BookDto` peut être peuplé depuis `BookEntity`. Les noms de champs sont mis en correspondance par convention ; les inadéquations sont déclarées explicitement :

```php
#[Map(from: BookEntity::class, properties: ['authorName' => 'author.fullName'])]
class BookDto {}
```

La notation pointée traverse les objets imbriqués. Le mapping s'exécute avant la sérialisation et remplace le comportement implicite de correspondance de propriétés du serializer. Les champs non mappés lèvent une erreur au moment de la configuration, pas à l'exécution.

ObjectMapper fonctionne avec `stateOptions` :

```php
use ApiPlatform\Doctrine\Orm\State\Options;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource(
    operations: [
        new GetCollection(
            stateOptions: new Options(entityClass: BookEntity::class),
        ),
    ]
)]
#[Map(from: BookEntity::class)]
class BookDto {}
```

Le provider récupère les objets `BookEntity` depuis Doctrine. ObjectMapper les convertit en instances `BookDto`. Le serializer écrit le DTO. Trois étapes distinctes, chacune avec un contrat clair.

## Intégration TypeInfo dans toute la pile

Symfony 7.1 a introduit le [composant TypeInfo](https://symfony.com/doc/current/components/type_info.html), une couche unifiée d'introspection des types qui comprend les types union, intersection, les collections génériques et les types nullable à travers la reflection, PHPDoc et la syntaxe PHP 8.x.

La 4.2 remplace la résolution de types interne d'API Platform par TypeInfo. Cela affecte la génération de schémas de filtres, l'inférence de schéma OpenAPI, et la coercition de types du serializer. Le bénéfice visible est que les types qui généraient auparavant des schémas OpenAPI incorrects ou manquants — `Collection<int, Book>`, `list<string>`, types intersection — produisent maintenant des schémas précis sans annotations `@ApiProperty` manuelles.

## Autoconfigure `#[ApiResource]`

Avant la 4.2, ajouter `#[ApiResource]` à une classe suffisait pour qu'Hugo la découvre uniquement si la classe était dans un chemin scanné par le chargeur de ressources d'API Platform. En dehors de ce chemin, une configuration de service explicite était nécessaire.

La 4.2 se branche sur le système d'autoconfigure de Symfony. Toute classe taguée avec `#[ApiResource]` est automatiquement enregistrée comme ressource indépendamment de son emplacement, tant qu'elle est dans un répertoire couvert par le scan de composants de Symfony. Aucune entrée dans `config/services.yaml` nécessaire.

Pour Laravel, l'équivalent utilise l'autoloading du service provider de Laravel — les modèles Eloquent avec `#[ApiResource]` sont récupérés automatiquement sans enregistrement manuel.

## Nouveaux filtres Doctrine de recherche

La 4.2 ajoute deux filtres Doctrine qui étaient couramment réimplémentés de zéro :

`ExistsFilter` — filtre une collection selon qu'une relation ou champ nullable est défini :

```php
#[ApiFilter(ExistsFilter::class, properties: ['publishedAt'])]
class Book {}
```

`GET /books?publishedAt[exists]=true` retourne les livres où `publishedAt` n'est pas null. `false` retourne les livres où il l'est.

`UuidFilter` — filtre par un champ UUID avec une gestion de type correcte pour les colonnes UUID :

```php
#[ApiFilter(UuidFilter::class, properties: ['externalId'])]
class Book {}
```

Aucun de ces filtres n'est nouveau — les deux patterns apparaissaient dans le code utilisateur depuis des années. La 4.2 les standardise pour qu'ils n'aient pas à être copiés entre les projets.
