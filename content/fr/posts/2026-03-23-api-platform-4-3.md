---
title: "API Platform 4.3 : serveur MCP, Scalar UI, et sécurité avant le provider"
date: 2026-03-23
series: ["api-platform-releases"]
part: 9
categories: [development]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.3 expose vos ressources à des agents IA via MCP, propose Scalar comme interface de documentation alternative, et évalue les expressions de sécurité avant la base de données."
translationKey: "api-platform-4-3"
---

API Platform 4.3 est sorti le 13 mars 2026. L'ajout principal est le support MCP : vos ressources API peuvent maintenant être exposées comme outils et ressources pour des agents LLM sans infrastructure supplémentaire. À côté de ça, deux améliorations pratiques sortent du lot : Scalar comme UI de documentation alternative, et les vérifications de sécurité qui s'exécutent avant que le state provider soit appelé.

## Support du serveur MCP

Le Model Context Protocol est le standard émergent pour connecter des LLMs à des outils et sources de données externes. 4.3 embarque une intégration MCP expérimentale qui se mappe directement sur le modèle de ressources d'API Platform.

Deux nouveaux attributs dans `ApiPlatform\Metadata` gèrent l'exposition :

**`McpTool`** marque une opération comme outil MCP qu'un agent peut appeler :

```php
use ApiPlatform\Metadata\McpTool;
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\GetCollection;

#[ApiResource]
#[McpTool(name: 'list_books', description: 'Returns a list of available books')]
#[GetCollection]
class Book {}
```

**`McpResource`** expose une ressource comme contenu MCP adressable, identifié par une URI :

```php
use ApiPlatform\Metadata\McpResource;

#[ApiResource]
#[McpResource(uri: 'books://{id}', mimeType: 'application/ld+json')]
class Book {}
```

Les deux attributs sont répétables et se composent avec les opérations API Platform existantes. Le format du serveur MCP est configurable :

```yaml
api_platform:
    mcp:
        format: 'jsonld'
```

L'intégration vit dans un package séparé `api-platform/mcp`. Les collections sont supportées. Symfony et Laravel sont couverts.

## Scalar comme UI de documentation

Swagger UI est l'interface de documentation par défaut depuis API Platform 2.x. 4.3 ajoute Scalar comme alternative opt-in. Scalar affiche les specs OpenAPI avec une interface plus moderne et une meilleure lisibilité pour les grandes APIs.

À activer dans la config (nécessite TwigBundle) :

```yaml
api_platform:
    enable_scalar: true
    openapi:
        scalar_extra_configuration:
            theme: 'purple'
            darkMode: true
```

## Sécurité avant le provider

Avant 4.3, l'expression `security:` d'une opération était évaluée après que le state provider avait déjà chargé l'objet depuis la base. Une requête non autorisée déclenchait quand même une requête Doctrine avant d'être rejetée. À grande échelle, ça comptait.

4.3 enregistre un nouveau décorateur `AccessCheckerProvider` avec une priorité 10, avant l'étape de lecture :

```php
#[ApiResource]
#[Get(security: "is_granted('ROLE_ADMIN')")]
class Book {}
```

Le contrôle `is_granted` s'exécute maintenant avant toute requête base de données. La contrepartie : `$object` est `null` au stade `pre_read`. Les expressions qui dépendent de l'objet chargé — `security: "object.getOwner() == user"` — doivent passer à `securityPostDenormalize`, qui s'exécute toujours après le provider.

C'est un changement comportemental. À vérifier sur vos expressions de sécurité si elles référencent `object`.

## ComparisonFilter

Un nouveau décorateur `ComparisonFilter` dans `ApiPlatform\Doctrine\Orm\Filter` enveloppe n'importe quel filtre d'égalité et ajoute les opérateurs de comparaison (`gt`, `gte`, `lt`, `lte`, `ne`) :

```php
use ApiPlatform\Doctrine\Orm\Filter\ComparisonFilter;
use ApiPlatform\Doctrine\Orm\Filter\RangeFilter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(filter: new ComparisonFilter(new RangeFilter()), property: 'price')]
class Product {}
```

Les paramètres OpenAPI pour chaque variante d'opérateur sont générés automatiquement. Le filtre est marqué expérimental.

## UuidFilter et support des relations imbriquées

Un nouveau `UuidFilter` dans `ApiPlatform\Doctrine\Orm\Filter` filtre les collections par propriétés UUID, y compris les relations imbriquées. L'`IriFilter` reçoit le même support dans cette version.

```php
use ApiPlatform\Doctrine\Orm\Filter\UuidFilter;
use ApiPlatform\Metadata\ApiFilter;

#[ApiResource]
#[ApiFilter(UuidFilter::class, properties: ['author.uuid'])]
class Book {}
```

## PartialSearchFilter avec contrôle de la casse

Le nouveau `PartialSearchFilter` dans `ApiPlatform\Doctrine\Orm\Filter` effectue une recherche partielle sur les chaînes. Par défaut, il encapsule la comparaison dans `LOWER()` pour des résultats insensibles à la casse. Passer `caseSensitive: true` désactive ce comportement :

```php
use ApiPlatform\Doctrine\Orm\Filter\PartialSearchFilter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(filter: new PartialSearchFilter(caseSensitive: true), property: 'title')]
class Book {}
```

## SortFilter ODM avec support des propriétés imbriquées

Un `SortFilter` arrive pour MongoDB (ODM) dans `ApiPlatform\Doctrine\Odm\Filter`, conçu exclusivement pour être utilisé avec le système `Parameters`. Il supporte les propriétés imbriquées nativement :

```php
use ApiPlatform\Doctrine\Odm\Filter\SortFilter;
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[QueryParameter(filter: new SortFilter(), property: 'department.name')]
class Employee {}
```

## Les entités Doctrine readonly perdent PUT et PATCH automatiquement

Si une entité Doctrine est déclarée en lecture seule avec `#[ORM\Entity(readOnly: true)]`, API Platform supprime maintenant automatiquement les opérations `PUT` et `PATCH` de sa définition de ressource. Pas besoin de liste d'opérations manuelle :

```php
use ApiPlatform\Metadata\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ApiResource]
#[ORM\Entity(readOnly: true)]
class AuditLog {}
```

`GET`, `GetCollection` et `DELETE` restent disponibles. Seules les opérations d'écriture sont retirées.

## Préfixe de namespace pour les makers

Le bundle Symfony accepte une nouvelle clé de configuration qui préfixe toutes les classes générées par les makers d'API Platform :

```yaml
api_platform:
    maker:
        namespace_prefix: 'Api'
```

Avec ce réglage, `make:api-resource` génère les classes sous `App\Api\` au lieu de `App\`.

## Valeurs par défaut sur QueryParameter

`QueryParameter` accepte maintenant une option `default` qui fournit une valeur de repli quand le paramètre est absent de la requête :

```php
use ApiPlatform\Metadata\QueryParameter;

#[ApiResource]
#[GetCollection]
#[QueryParameter(key: 'order', default: 'asc')]
class Book {}
```

## Validation UUID et ULID sur les paramètres

Les paramètres de requête et headers déclarés avec un format `uuid` ou `ulid` sont maintenant validés automatiquement. Les identifiants mal formés sont rejetés avant d'atteindre le state provider, sans contrainte supplémentaire à ajouter.

## ProblemExceptionInterface

Les exceptions personnalisées peuvent maintenant implémenter `ProblemExceptionInterface` pour mapper directement vers les champs [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) Problem Details (`type`, `title`, `detail` et membres d'extension). Auparavant, cela nécessitait un normalizer d'exception personnalisé.

## Headers LDP

API Platform ajoute maintenant automatiquement les headers de réponse `Accept-Post` et `Allow` sur les endpoints de collection et d'item, conformément à la [spécification Linked Data Platform](https://www.w3.org/TR/ldp/). Cela améliore la découvrabilité pour les clients hypermedia qui s'appuient sur ces headers pour déterminer quelles opérations sont disponibles sur une ressource donnée.

## Breaking changes

Quelques changements comportementaux à connaître avant de migrer :

- **La sécurité s'exécute avant le provider.** Les expressions qui référencent `object` recevront `null`. À déplacer vers `securityPostDenormalize`.
- **Les filtres Doctrine requièrent un `property` explicite.** Un attribut `property` manquant lève maintenant une exception au lieu d'échouer silencieusement.
- **Le `@type` JSON-LD avec `output` et `itemUriTemplate`** utilise un nom de classe différent pour être sémantiquement cohérent.
