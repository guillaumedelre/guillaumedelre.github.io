---
title: "API Platform 4.1 : paramètres de requête stricts, OpenAPI multi-spec, et limites GraphQL"
date: 2025-02-28
series: ["api-platform-releases"]
part: 7
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.1 formalise la validation stricte des paramètres de requête, introduit x-apiplatform-tag pour découper une API en plusieurs specs OpenAPI, et ajoute des limites de profondeur et complexité pour GraphQL."
---

API Platform 4.1 est arrivé en février 2025 avec un lot de fonctionnalités moins axées sur de nouvelles capacités que sur la mise en production des existantes. La validation stricte des paramètres de requête gagne une propriété de première classe. OpenAPI gagne un mécanisme pour découper les grandes APIs en specs séparées. GraphQL obtient les contrôles de prévention des abus qui lui manquaient.

## Validation stricte des paramètres de requête

La 3.3 avait introduit la validation des paramètres de requête en opt-in. La 3.4 avait déprécié le comportement permissif. La 4.1 la formalise avec une propriété native `strictQueryParameterValidation` sur les ressources et opérations : quand elle est à `true`, les paramètres de requête inconnus renvoient 400.

```php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\Metadata\QueryParameter;

#[GetCollection(
    strictQueryParameterValidation: true,
    parameters: [
        new QueryParameter(key: 'utm_source', required: false, schema: ['type' => 'string']),
        new QueryParameter(key: 'feature_flag', required: false, schema: ['type' => 'string']),
    ]
)]
class Book {}
```

Les paramètres déclarés passent ; les paramètres non déclarés sont rejetés. Pour désactiver la validation stricte sur une opération spécifique quand elle est activée au niveau de la ressource, passer `strictQueryParameterValidation: false` sur cette opération.

## `x-apiplatform-tag` pour OpenAPI multi-spec

Les grandes APIs ont souvent besoin de plusieurs specs OpenAPI : une par équipe, une par version d'API, une interne et une publique. Avant la 4.1, la spec générée était un seul document, et la diviser nécessitait un post-traitement ou des instances API Platform séparées.

La 4.1 ajoute une extension vendor `x-apiplatform-tag` (sans `s` final). On tague les opérations avec des noms de groupes logiques via les `extensionProperties` d'un objet `Operation` OpenAPI, puis on demande la spec filtrée sur un ou plusieurs groupes :

```php
use ApiPlatform\Metadata\GetCollection;
use ApiPlatform\OpenApi\Factory\OpenApiFactory;
use ApiPlatform\OpenApi\Model\Operation;

#[GetCollection(
    openapi: new Operation(
        extensionProperties: [OpenApiFactory::API_PLATFORM_TAG => ['public', 'v2']]
    )
)]
class Book {}
```

Demander `/api/docs.json?filter_tags[]=public` retourne uniquement les opérations taguées `public`. La spec complète reste disponible sans filtre. Les groupes n'affectent pas le comportement réel de l'API — c'est uniquement une préoccupation de couche documentation.

Cela rend faisable de maintenir une configuration API Platform unique tout en servant différentes vues de spec à différents consommateurs : un Swagger UI public, un portail partenaire, et un outil interne qui expose les endpoints d'administration.

## Authentification HTTP dans Swagger UI

Avant la 4.1, le Swagger UI fourni avec API Platform supportait l'authentification par token Bearer via son dialogue "Authorize". L'authentification par clé API et HTTP Basic n'étaient pas câblées.

La 4.1 ajoute le support de plusieurs schémas de sécurité dans le document OpenAPI généré. Les schémas de sécurité sont ajoutés en décorant l'`OpenApiFactory` et en modifiant l'objet `components.securitySchemes` de la spec. Chaque schéma déclaré apparaît ensuite dans le dialogue "Authorize" de Swagger UI et est appliqué aux requêtes faites depuis l'UI. C'est une amélioration de la documentation et de l'expérience développeur — la logique d'authentification réelle dans votre application n'est pas affectée.

## Limites de profondeur et complexité des requêtes GraphQL

La structure de requête récursive de GraphQL rend triviale la construction d'une requête petite en octets mais énorme en coût d'exécution. Sans limites, une requête imbriquée sur quatre niveaux à travers une relation many-to-many peut frapper la base de données des centaines de fois.

La 4.1 ajoute des limites configurables de profondeur et complexité :

```yaml
api_platform:
    graphql:
        max_query_depth: 10
        max_query_complexity: 100
```

`max_query_depth` est le niveau d'imbrication maximum. `max_query_complexity` assigne un coût à chaque champ et rejette les requêtes dont le coût total dépasse le seuil. Les requêtes qui dépassent l'une ou l'autre limite sont rejetées avant exécution avec une réponse 400.

Il n'y a pas de valeur universellement correcte pour ces limites — elles dépendent de la forme de votre schéma et des patterns de requête attendus. Les valeurs par défaut sont intentionnellement permissives pour éviter de casser des APIs existantes lors de la mise à jour. Les resserrer est un choix de configuration délibéré.

## Formats de sortie au niveau opération

La 4.0 et les versions antérieures configuraient les types de contenu acceptés et retournés au niveau de l'API. La 4.1 permet de les restreindre par opération :

```php
use ApiPlatform\Metadata\GetCollection;

#[GetCollection(
    outputFormats: ['jsonld' => ['application/ld+json']],
    inputFormats: ['json' => ['application/json']],
)]
class Book {}
```

Les opérations qui ne spécifient pas de formats héritent de la configuration au niveau de l'API. C'est utile pour les endpoints qui doivent retourner un format spécifique (une export CSV, un flux binaire) sans changer les défauts pour le reste de l'API.
