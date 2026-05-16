---
title: "API Platform 3.1 : votre ressource n'a pas à être votre entité"
date: 2023-01-23
series: ["api-platform-releases"]
part: 2
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.1 découple les ressources API des entités Doctrine, introduit un PUT conforme à la spec, et collecte les erreurs de dénormalisation en liste."
---

Deux mois après la 3.0, API Platform 3.1 est arrivé avec le premier lot de fonctionnalités construites sur le nouveau modèle d'état. Tous les changements ne sont pas spectaculaires, mais l'un d'eux résout un problème qui a engendré beaucoup de contournements alambiqués en 2.x : votre ressource API n'a plus besoin d'être votre entité Doctrine.

## La séparation ressource/entité

En 2.x, API Platform fonctionnait mieux quand votre ressource API et votre modèle de persistance étaient la même classe. Utiliser un DTO comme surface API était possible via le système Input/Output DTO, mais ce système a été supprimé en 3.0 — il compliquait le modèle d'état sans apporter suffisamment de bénéfices.

La 3.1 le remplace par quelque chose de plus propre. Le paramètre `stateOptions` d'une opération accepte un objet `DoctrineOrmOptions` qui pointe vers une entité différente :

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

Le provider reçoit les objets `BookEntity` de Doctrine et la couche de sérialisation les mappe sur `BookDto`. Les filtres Doctrine, la pagination et le tri fonctionnent tous sur `BookEntity`. La surface API expose `BookDto`. Les deux peuvent évoluer indépendamment.

Ça compte plus qu'il n'y paraît. Votre modèle de persistance accumule des champs internes, des relations et des colonnes qui n'ont aucune raison d'apparaître dans votre API. Avant la 3.1, on les exposait quand même ou on construisait un normaliseur élaboré pour les cacher. Maintenant on déclare ce que l'API expose comme classe distincte et on laisse le framework gérer la correspondance.

## Un PUT conforme à la spec

Depuis la version 1.0, le handler PUT d'API Platform mettait à jour des ressources existantes. Créer une ressource via PUT — ce que la spec HTTP autorise explicitement — n'était pas supporté. La 3.1 ajoute la création basée sur l'`uriTemplate` :

```php
#[Put(
    uriTemplate: '/books/{id}',
    allowCreate: true,
)]
```

Avec `allowCreate: true`, un PUT vers une URI qui n'existe pas crée la ressource au lieu de retourner une 404. L'identifiant vient de l'URI, pas du corps de la requête. C'est ce que la RFC 9110 décrit pour PUT : "Si la ressource cible n'a pas de représentation courante et que le PUT en crée une avec succès, le serveur d'origine DOIT informer le user agent en envoyant une réponse 201 (Created)."

C'est un petit paramètre, mais il ouvre API Platform à des cas d'usage — création idempotente, identifiants assignés par le client — qui nécessitaient auparavant un contrôleur personnalisé.

## Les erreurs de dénormalisation collectées, pas jetées

Avant la 3.1, les erreurs de désérialisation s'arrêtaient au premier problème. Envoyer un corps de requête avec cinq champs invalides renvoyait une erreur sur le premier. On le corrigeait, on renvoyait, on trouvait le deuxième. À répéter cinq fois.

La 3.1 ajoute une option `collect_denormalization_errors` sur l'opération qui change ce comportement :

```php
#[Post(collectDenormalizationErrors: true)]
```

Avec cette option activée, API Platform intercepte toutes les erreurs de type et les violations de contraintes pendant la désérialisation et les retourne sous forme de liste structurée dans la réponse, formatée de la même façon que les erreurs de validation. Un seul aller-retour, une vue complète.

## `ApiResource::openapi` remplace `openapiContext`

L'ancien paramètre `openapiContext` acceptait un tableau brut fusionné dans le schéma OpenAPI généré — pratique mais non typé. La 3.1 introduit un paramètre `openapi` de premier ordre qui accepte un objet `OpenApiOperation` :

```php
use ApiPlatform\OpenApi\Model\Operation;
use ApiPlatform\OpenApi\Model\RequestBody;

#[Post(
    openapi: new Operation(
        requestBody: new RequestBody(
            description: 'Créer un livre',
            required: true,
        ),
        summary: 'Créer une nouvelle entrée livre',
    )
)]
```

L'ancien tableau `openapiContext` fonctionne encore mais est déprécié. La nouvelle approche est typée, compatible avec l'autocomplétion IDE, et se valide à la construction plutôt qu'à la génération du schéma. Les backed enums PHP 8.1 obtiennent également une génération de schéma OpenAPI correcte en 3.1 — un champ typé comme backed enum produit un schéma avec des valeurs `enum` et le type correct, sans annotation supplémentaire.

## Le pattern est clair

La 3.0 a établi l'architecture. La 3.1 montre ce que cette architecture permet : une séparation propre ressource/entité sans système DTO parallèle, une sémantique HTTP correcte selon la RFC, un meilleur rapport d'erreurs. Aucune de ces fonctionnalités n'aurait été aussi propre à implémenter sur le modèle data provider de la 2.x. Les features de la 3.1 sont la première preuve que la réécriture était la bonne décision.
