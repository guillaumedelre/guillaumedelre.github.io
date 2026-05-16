---
title: "API Platform 4.0 : support Laravel et PUT repensé"
date: 2024-09-27
series: ["api-platform-releases"]
part: 6
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 4.0 apporte le support Laravel de première classe avec Eloquent et les policies, et supprime PUT des opérations par défaut pour corriger une ambiguïté sémantique persistante."
---

API Platform 4.0 est sorti neuf jours après la 3.4, fin septembre 2024. Le numéro de version est honnête : il n'y a pas de nouvelle architecture, et la migration depuis la 3.4 est courte si on a résolu les dépréciations. Ce qui en fait un majeur, c'est le changement de scope — API Platform n'est plus un framework uniquement Symfony — et un défaut d'opinion qui inverse six ans de comportement PUT.

## Laravel comme cible de première classe

Depuis sa première release, API Platform était construit sur Symfony. La couche HTTP, les métadonnées, le serializer et le bridge Doctrine supposaient tous le container de Symfony, l'event dispatcher et le cycle de vie des requêtes. Les utilisateurs Laravel pouvaient faire tourner API Platform via un adaptateur léger, mais les filtres, la sécurité et l'intégration Doctrine ne fonctionnaient pas avec Eloquent.

La 4.0 livre un bridge Laravel dédié. Il mappe la couche d'état d'API Platform sur le cycle de vie des requêtes de Laravel, s'intègre directement avec les modèles Eloquent, et se branche sur le système d'autorisation de Laravel :

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\GetCollection;
use Illuminate\Database\Eloquent\Model;

#[ApiResource(
    operations: [new GetCollection(), new Get()]
)]
class Book extends Model
{
    protected $fillable = ['title', 'author'];
}
```

L'autorisation utilise les policies et gates de Laravel plutôt que les security voters de Symfony :

```php
#[Get(security: "policy('view', object)")]
class Book extends Model {}
```

L'expression `policy()` appelle `Gate::check()` de Laravel avec l'instance du modèle. Les filtres Doctrine Orm — `SearchFilter`, `RangeFilter`, `OrderFilter` — sont portés sur Eloquent. La pagination, le tri et la validation fonctionnent via les mécanismes natifs de Laravel.

Ce n'est pas un shim de compatibilité. Le bridge Laravel est maintenu aux côtés du bridge Symfony et est couvert par la même suite de tests. Les projets utilisant l'un ou l'autre framework ont la même API de définition des ressources.

## PUT supprimé des opérations par défaut

Depuis API Platform 1.0, `#[ApiResource]` sans tableau `operations` explicite générait des opérations CRUD incluant PUT. Le handler PUT mettait à jour les ressources existantes et, depuis la 3.1, pouvait aussi les créer via `allowCreate: true`.

La 4.0 supprime PUT de l'ensemble par défaut. `#[ApiResource]` génère maintenant GET, POST, PATCH et DELETE. Pour utiliser PUT, il faut le déclarer explicitement :

```php
use ApiPlatform\Metadata\ApiResource;
use ApiPlatform\Metadata\Put;

#[ApiResource(
    operations: [
        // ... autres opérations
        new Put(),
    ]
)]
class Book {}
```

La motivation est la clarté sémantique. PATCH remplace PUT pour la plupart des cas d'usage de mise à jour partielle. La sémantique de PUT — remplacer la représentation entière de la ressource — est rarement ce qu'une API implémente réellement, mais le défaut le faisait apparaître dans toutes les APIs à moins de l'enlever activement. Rendre PUT opt-in aligne les défauts sur la façon dont la sémantique HTTP est réellement utilisée en pratique.

## PHP 8.2 minimum

La 4.0 abandonne PHP 8.0 et 8.1. PHP 8.2 est le nouveau minimum. La syntaxe de classe readonly, `AllowDynamicProperties` et les types DNF[^dnf] introduits en 8.2 sont disponibles dans toute la codebase. Aucune fonctionnalité spécifique de 8.2 n'est structurante pour la 4.0 — le bump de version concerne principalement l'abandon du fardeau de maintenance plus ancien.

## Symfony 7 et Doctrine ORM 3 minimum

Côté Symfony, la 4.0 requiert Symfony 7.0 et Doctrine ORM 3. Les deux étaient déjà supportés en 3.4. La migration de la 3.4 vers la 4.0 sur la piste Symfony est : résoudre les dépréciations 3.4, vérifier qu'on est sur Symfony 7 et ORM 3, puis mettre à jour. Aucun travail de migration supplémentaire n'est nécessaire si c'est déjà en place.

## Ce que la 4.0 n'est pas

La 4.0 n'est pas une nouvelle architecture. Les state providers, processors et le modèle de métadonnées de ressources de la 3.0 sont inchangés. Le bridge Laravel ajoute un nouveau contexte d'exécution mais ne change pas la façon dont les ressources ou les opérations sont déclarées. La séparation est intentionnelle : si la 3.0 était le "quoi", la 4.0 est le "où".

[^dnf]: Types en Forme Normale Disjonctive : types intersection combinés avec union, comme `(A&B)|null`.
