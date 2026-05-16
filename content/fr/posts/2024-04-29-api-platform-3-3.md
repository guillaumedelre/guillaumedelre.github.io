---
title: "API Platform 3.3 : headers, sécurité des liens, et webhooks OpenAPI"
date: 2024-04-29
series: ["api-platform-releases"]
part: 4
categories: [développement]
tags: [api-platform, php, symfony, openapi]
description: "API Platform 3.3 ajoute la configuration déclarative des headers, la sécurité fine sur les liens de sous-ressources, et le support des webhooks OpenAPI."
---

API Platform 3.3 est sorti en avril 2024 avec un ensemble d'ajouts ciblés. Aucun d'eux ne remodèle l'architecture — la 3.2 avait déjà clos ce chapitre. Ce que la 3.3 apporte, c'est du contrôle sur des choses qui étaient soit codées en dur soit nécessitaient un contournement : les headers de réponse, la visibilité des liens sur les sous-ressources, et les webhooks dans la spec générée.

## Configuration déclarative des headers

Avant la 3.3, définir des headers de réponse personnalisés nécessitait soit un processor personnalisé qui modifiait l'objet réponse, soit un event listener Symfony sur `kernel.response`. Les deux approches fonctionnaient mais vivaient en dehors de la définition de la ressource.

La 3.3 ajoute un paramètre `headers` aux métadonnées d'opération :

```php
use ApiPlatform\Metadata\Get;
use ApiPlatform\Metadata\HeaderParameter;

#[Get(
    headers: [
        new HeaderParameter(name: 'X-Custom-Header', description: 'A custom header'),
    ]
)]
```

Pour les headers qui varient par réponse (comme `Cache-Control` avec un max-age calculé), le processor peut encore les définir directement sur l'objet réponse. Le paramètre `headers` sert principalement à documenter les headers attendus dans la spec OpenAPI et pour les valeurs de headers statiques.

## Sécurité des liens sur les sous-ressources

Quand une ressource expose des liens vers des ressources liées, ces liens apparaissent dans la sortie sérialisée indépendamment du fait que l'utilisateur courant puisse accéder à la ressource liée. Cela crée un problème de divulgation : un utilisateur qui peut lire un livre mais pas le profil de son auteur voit quand même l'URI de l'auteur dans la réponse.

La 3.3 ajoute des expressions de sécurité au descripteur `Link` :

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

Le lien est omis de la réponse quand l'expression de sécurité évalue à false. La ressource liée elle-même n'est pas affectée — seulement le fait que la réponse courante inclue la référence à elle.

## `ApiProperty::security`

Le même mécanisme d'expression de sécurité est disponible au niveau propriété via `ApiProperty::security`. Cela permet de cacher des champs individuels selon l'utilisateur courant sans écrire un normalizer personnalisé :

```php
use ApiPlatform\Metadata\ApiProperty;

class Book
{
    #[ApiProperty(security: "is_granted('ROLE_ADMIN')")]
    public string $internalNote;
}
```

La propriété est exclue de la sérialisation quand l'expression est false. C'est plus propre qu'un normalizer pour le cas courant de champs conditionnels par rôle.

## Webhooks OpenAPI

[OpenAPI 3.1](https://spec.openapis.org/oas/v3.1.0) supporte les webhooks — des appels HTTP sortants que votre API fait à des listeners enregistrés — dans le document spec lui-même. Avant la 3.3, il n'y avait pas de moyen de les documenter dans la spec générée par API Platform.

La 3.3 ajoute une option de configuration `webhooks` où vous pouvez déclarer la forme de vos appels sortants :

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

Les définitions de webhooks apparaissent dans la spec générée aux côtés des paths réguliers. Swagger UI les affiche dans une section séparée.

## Deep linking dans Swagger UI

Swagger UI supporte le deep linking — des URLs mémorisables qui ouvrent directement sur une opération spécifique dans l'interface. Avant la 3.3, l'intégration API Platform n'activait pas cela. La 3.3 active l'option `deepLinking` de Swagger UI :

```yaml
api_platform:
    swagger_ui:
        deep_linking: true
```

Avec cette option activée, le fragment d'URL se met à jour pendant la navigation dans l'UI, et coller ou partager l'URL ouvre la même opération. Utile quand on écrit de la doc qui pointe directement vers un endpoint spécifique.

## Validation stricte des paramètres de requête

La 3.3 renforce le validateur de paramètres de requête : les paramètres non déclarés sur l'opération renvoient maintenant une réponse 400 au lieu d'être silencieusement ignorés. Ce comportement est opt-in :

```yaml
api_platform:
    validator:
        query_parameter_validation: true
```

L'intention est de détecter les fautes de frappe et les mauvaises utilisations de l'API tôt. Si vous vous appuyez sur des paramètres de requête pass-through pour une logique personnalisée (logging, feature flags), vous devez les déclarer explicitement sur l'opération avant d'activer cela.
