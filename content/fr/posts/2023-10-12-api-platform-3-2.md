---
title: "API Platform 3.2 : les erreurs comme ressources et le retour des sous-ressources"
date: 2023-10-12
series: ["api-platform-releases"]
part: 3
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.2 fait des erreurs des ressources Problem Detail de première classe, rétablit les sous-ressources proprement, et rend les event listeners optionnels."
---

API Platform 3.2 est arrivé en octobre 2023 avec trois changements qui ont fait avancer le modèle d'état : les erreurs sont devenues des ressources, les sous-ressources sont revenues sous une forme qui s'intègre vraiment dans l'architecture, et le dernier point d'extension hérité — les event listeners — a été formellement remplacé.

## Les erreurs comme ressources

Avant la 3.2, la gestion des erreurs était en dehors du modèle de ressources. Les exceptions étaient interceptées par un event listener Symfony et converties en réponse, avec un contrôle limité sur la forme de la sortie.

La 3.2 fait des erreurs des classes `ApiResource` de première classe conformes à la [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) (Problem Details for HTTP APIs). La classe d'erreur intégrée est `ApiPlatform\ApiResource\Error`, et on peut créer la sienne :

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

Quand cette exception est levée n'importe où dans la couche d'état, API Platform l'intercepte, la sérialise comme une réponse Problem Detail, et génère un schéma OpenAPI approprié pour elle. Le type d'erreur, le titre, le détail et le statut font tous partie du contrat de la ressource — pas des chaînes codées en dur dans un listener.

## Les sous-ressources sans les contournements

Les sous-ressources existaient en 2.x mais ont été supprimées en 3.0 parce qu'elles étaient étroitement couplées à l'ancien modèle de data provider et ne pouvaient pas être proprement mappées à la nouvelle architecture orientée opération. La 3.2 les réintroduit d'une façon qui s'intègre.

Une sous-ressource est une ressource accessible via l'URI d'une ressource parente. En 3.2, elle est déclarée directement sur la ressource enfant en utilisant `uriTemplate` :

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

Le descripteur `Link` rend la relation explicite. Le provider reçoit `bookId` dans `$uriVariables` et peut l'utiliser pour délimiter la requête. Pas d'inférence magique, pas de jointures implicites — la structure d'URI et l'accès aux données sont tous les deux déclarés.

## `canonical_uri_template` pour plusieurs chemins d'accès

Quand une ressource est accessible via plusieurs URI (un endpoint direct et un endpoint de sous-ressource), OpenAPI doit savoir quelle URI est canonique pour les liens `$ref`. La 3.2 utilise le `uriTemplate` de niveau supérieur sur `ApiResource` comme URI canonique par défaut. Pour plus de contrôle, l'option `canonical_uri_template` peut être passée via `extraProperties` sur n'importe quelle opération pour la définir explicitement.

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

La spec OpenAPI générée utilise l'URI canonique pour les références de schéma, gardant le document cohérent quand une ressource apparaît sous plusieurs chemins.

## Types union et intersection

La 3.2 ajoute le support des types union et intersection PHP dans la couche de métadonnées. Une propriété déclarée comme `Book|Magazine` génère un schéma `oneOf` approprié dans OpenAPI. C'était auparavant non supporté — on devait tomber sur un `mixed` non typé ou annoter la propriété manuellement.

## Les event listeners rendus optionnels

Le dernier shim de compatibilité venu de la 2.x était la possibilité d'utiliser les event listeners Symfony sur les événements `kernel.request` et `kernel.view` pour intercepter le flux de données d'API Platform. La 3.2 ne les supprime pas, mais introduit un moyen de s'en passer : passer `event_listeners_backward_compatibility_layer: false` dans la configuration d'API Platform désactive entièrement les hooks basés sur les événements. Le remplacement est un provider ou processor décoré par un autre provider ou processor. Le hook basé sur les événements était avec état, dépendant de l'ordre, et court-circuitait entièrement le contexte d'opération. Les providers décorés reçoivent l'objet opération et peuvent appeler le provider interne quand ils sont prêts.

## Le modèle d'état est maintenant complet

La 3.0 a introduit l'architecture. La 3.1 a ajouté la séparation ressource/entité. La 3.2 ferme les lacunes restantes : les erreurs ont un contrat de ressource, les sous-ressources ont un modèle de déclaration propre, et la couche d'état couvre désormais tous les points d'extension que les event listeners géraient autrefois. Les shims 2.x existent encore, mais s'en passer n'est plus qu'une ligne de configuration.
