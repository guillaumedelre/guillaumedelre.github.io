---
title: "La recherche full-text PostgreSQL avec Doctrine, sans une ligne de SQL brut"
date: 2025-02-10
categories: [développement]
tags: [doctrine, postgresql, symfony, php, api-platform, recherche-plein-texte]
description: "Comment nous avons superposé des types DBAL personnalisés et des wrappers DQL sur postgresql-for-doctrine pour intégrer la recherche full-text PostgreSQL dans un projet Symfony API Platform."
---

Le champ de recherche de la médiathèque renvoyait des résultats en 800 millisecondes en staging. En production, il y avait quarante fois plus de lignes. Le plan d'exécution révélait un sequential scan: aucun index sollicité, aucune façon d'y remédier avec un B-tree classique. L'équipe produit voulait aussi une recherche multi-mots: taper "interview président", obtenir des résultats contenant les deux termes. Une requête `LIKE` avec des wildcards n'a pas de manière propre d'exprimer ça sans conditions indépendantes multiples, chacune nécessitant son propre scan.

PostgreSQL embarque une recherche full-text depuis plus de quinze ans. La plateforme tournait déjà sous PostgreSQL. Le hic: le projet utilise Doctrine ORM, et Doctrine ne sait pas nativement ce qu'est un `tsvector`.

Une bibliothèque communautaire, <a href="https://github.com/martin-georgiev/postgresql-for-doctrine" target="_blank" rel="noopener noreferrer">postgresql-for-doctrine</a>, couvre une partie de cette lacune. Elle enregistre des fonctions DQL basiques comme `TO_TSQUERY`, `TO_TSVECTOR`, et l'opérateur de correspondance `@@` en tant que pièces atomiques séparées. La fondation était là. Trois choses restaient à construire par-dessus.

## Le type que Doctrine n'a jamais vu

<a href="https://www.postgresql.org/docs/current/datatype-textsearch.html" target="_blank" rel="noopener noreferrer">La recherche full-text de PostgreSQL</a> repose sur deux types: `tsvector` (une liste pré-traitée de tokens normalisés) et `tsquery` (une expression de recherche). On maintient une colonne `tsvector`, on l'indexe avec GIN, et on interroge avec l'opérateur `@@`.

Le DBAL de Doctrine ne livre aucun type `tsvector`. Déclarer `#[ORM\Column(type: 'tsvector')]` sans l'enregistrer au préalable lève une `UnknownColumnTypeException`. La solution: un type DBAL personnalisé:

```php
class TsVector extends Type
{
    final public const string DBAL_TYPE = 'tsvector';

    public function getSQLDeclaration(array $column, AbstractPlatform $platform): string
    {
        return self::DBAL_TYPE;
    }

    public function getName(): string
    {
        return self::DBAL_TYPE;
    }

    public function convertToDatabaseValueSQL(string $sqlExpr, AbstractPlatform $platform): string
    {
        return sprintf("to_tsvector('simple', %s)", $sqlExpr);
    }

    public function convertToDatabaseValue(mixed $value, AbstractPlatform $platform): mixed
    {
        if (is_array($value) && isset($value['data'])) {
            return $value['data'];
        }

        return is_string($value) ? $value : null;
    }

    public function getMappedDatabaseTypes(AbstractPlatform $platform): array
    {
        return [self::DBAL_TYPE];
    }
}
```

La méthode intéressante est `convertToDatabaseValueSQL()`. Doctrine l'appelle pour envelopper le placeholder SQL avant que la valeur n'atteigne la base de données. La valeur écrite devient automatiquement `to_tsvector('simple', ?)` à la frontière DBAL, sans étape supplémentaire côté appelant.

On enregistre le type dans `doctrine.yaml`, puis on mappe la colonne sur l'entité:

```yaml
doctrine:
    dbal:
        types:
            tsvector: App\Doctrine\DBAL\Types\TsVector
```

```php
#[ORM\Column(type: 'tsvector', nullable: true)]
protected ?string $textSearch = null;
```

Côté PHP, la valeur est une simple chaîne. La conversion en vrai `tsvector` se fait invisiblement au niveau DBAL.

Nous avons utilisé le dictionnaire `'simple'`, qui tokenise sur les espaces et la ponctuation sans stemming spécifique à une langue. La plateforme gère plusieurs langues, et les règles de stemming français auraient cassé l'espagnol. Simple suffit largement pour la phonétique.

## Garder la colonne à jour

Une colonne `tsvector` est une donnée dérivée: elle doit rester synchronisée avec les champs source chaque fois que l'entité change. Un event listener Doctrine s'en charge:

```php
#[AsDoctrineListener(event: Events::prePersist)]
#[AsDoctrineListener(event: Events::preUpdate)]
class MediaTsVectorSubscriber
{
    public function prePersist(PrePersistEventArgs $event): void
    {
        if (!$event->getObject() instanceof Media) {
            return;
        }
        $this->updateTextSearch($event->getObject());
    }

    public function preUpdate(PreUpdateEventArgs $event): void
    {
        if (!$event->getObject() instanceof Media) {
            return;
        }
        $this->updateTextSearch($event->getObject());
    }

    private function updateTextSearch(Media $entity): void
    {
        $entity->setTextSearch(
            sprintf('%s %s', $entity->getTitle(), $entity->getCaption())
        );
    }
}
```

Avant chaque persist et update, le subscriber concatène les champs qui doivent être recherchables dans `textSearch`. Doctrine flush la chaîne combinée, le type DBAL l'enveloppe dans `to_tsvector('simple', ...)`, et PostgreSQL stocke la forme tokenisée.

Une subtilité: la valeur côté PHP est `"title caption"`, pas la sortie tsvector réelle. La base affiche `'caption' 'title'` (tokens triés), mais l'entité contient une chaîne brute. C'est attendu: la conversion est une responsabilité DBAL, pas PHP. Ça peut dérouter le débogage jusqu'à ce qu'on se souvienne où se situe la frontière.

## Étendre DQL avec les opérateurs FTS

Le DQL de Doctrine couvre les opérations SQL courantes, mais tout ce qui est spécifique à PostgreSQL est hors périmètre. C'est là que `postgresql-for-doctrine` entre en jeu: il enregistre `TO_TSQUERY`, `TO_TSVECTOR`, et `TSMATCH` comme fonctions DQL individuelles. Écrire une requête full-text en DQL sans lui signifierait basculer en SQL natif.

Les fonctions de la bibliothèque sont atomiques, cependant. Chacune correspond à un appel SQL. Exprimer une vérification de correspondance complète en DQL ressemble à `TSMATCH(o.textSearch, TO_TSQUERY(:term))`. Assez lisible, mais l'équipe voulait quelque chose de plus compact: une seule fonction DQL encodant à la fois l'opérateur de correspondance et le type de requête, y compris `websearch_to_tsquery` que `postgresql-for-doctrine` ne fournissait pas.

La solution: des <a href="https://www.doctrine-project.org/projects/doctrine-orm/en/latest/cookbook/dql-user-defined-functions.html" target="_blank" rel="noopener noreferrer">fonctions DQL personnalisées</a> via `FunctionNode`. On parse la syntaxe DQL, puis on émet du SQL. Toutes les fonctions FTS partagent la même signature à deux arguments, donc une classe abstraite de base gère le parsing:

```php
abstract class TsFunction extends FunctionNode
{
    public PathExpression|Node|null $ftsField = null;
    public PathExpression|Node|null $queryString = null;

    public function parse(Parser $parser): void
    {
        $parser->match(TokenType::T_IDENTIFIER);
        $parser->match(TokenType::T_OPEN_PARENTHESIS);
        $this->ftsField = $parser->StringPrimary();
        $parser->match(TokenType::T_COMMA);
        $this->queryString = $parser->StringPrimary();
        $parser->match(TokenType::T_CLOSE_PARENTHESIS);
    }
}
```

Chaque classe concrète implémente `getSql()` pour émettre son expression PostgreSQL:

```php
// e.textSearch @@ websearch_to_tsquery('simple', :term)
class TsWebsearchQueryFunction extends TsFunction
{
    public function getSql(SqlWalker $sqlWalker): string
    {
        return $this->ftsField->dispatch($sqlWalker)
            ." @@ websearch_to_tsquery('simple', "
            .$this->queryString->dispatch($sqlWalker).')';
    }
}

// ts_rank(e.textSearch, to_tsquery(:term)) pour le tri par pertinence
class TsRankFunction extends TsFunction
{
    public function getSql(SqlWalker $sqlWalker): string
    {
        return 'ts_rank('
            .$this->ftsField->dispatch($sqlWalker)
            .', to_tsquery('.$this->queryString->dispatch($sqlWalker).'))';
    }
}
```

```yaml
doctrine:
    orm:
        entity_managers:
            default:
                dql:
                    string_functions:
                        tswebsearchquery: App\Doctrine\ORM\Query\AST\Functions\TsWebsearchQueryFunction
                        tsrank: App\Doctrine\ORM\Query\AST\Functions\TsRankFunction
                        tsquery: App\Doctrine\ORM\Query\AST\Functions\TsQueryFunction
                        tsplainquery: App\Doctrine\ORM\Query\AST\Functions\TsPlainQueryFunction
```

`websearch_to_tsquery` est le bon choix pour la recherche côté utilisateur: les espaces deviennent des AND, les chaînes entre guillemets deviennent des phrases, `-mot` exclut un terme. Inutile d'apprendre aux utilisateurs à taper `interview & président`. C'est disponible depuis PostgreSQL 11. Sur les versions antérieures, `plainto_tsquery` est l'équivalent le plus proche.

## Le filtre API Platform et l'index GIN

Avec les fonctions DQL enregistrées, le filtre API Platform est simple. Un `AbstractFilter` personnalisé appelle directement la fonction DQL dans le `QueryBuilder`:

```php
class TextSearchFilter extends AbstractFilter
{
    protected function filterProperty(
        string $property,
        $value,
        QueryBuilder $queryBuilder,
        QueryNameGeneratorInterface $queryNameGenerator,
        string $resourceClass,
        ?Operation $operation = null,
        array $context = []
    ): void {
        if ('textSearch' !== $property || empty($value)) {
            return;
        }

        $queryBuilder
            ->andWhere('tswebsearchquery(o.textSearch, :value) = true')
            ->setParameter(':value', $value);
    }

    public function getDescription(string $resourceClass): array
    {
        return [];
    }
}
```

On l'applique sur l'entité avec la déclaration d'index:

```php
#[ORM\Index(
    columns: ['text_search'],
    name: 'media_text_search_idx_gin',
    options: ['USING' => 'gin (text_search)']
)]
#[ApiFilter(TextSearchFilter::class, properties: ['textSearch' => 'partial'])]
class Media
{
    // ...
    #[ORM\Column(type: 'tsvector', nullable: true)]
    protected ?string $textSearch = null;
}
```

L'option `USING gin` n'est pas négociable. Un index B-tree standard sur une colonne `tsvector` est inutile: PostgreSQL ne peut pas l'utiliser pour les requêtes `@@`. GIN (Generalized Inverted Index) fonctionne différemment: il indexe chaque token individuellement, donc les recherches par n'importe quel token sont en `O(log n)` plutôt que `O(n)`. Sans ça, on a construit un système qui donne l'impression d'être rapide mais qui fait quand même un full table scan.

Un `GET /media?textSearch=interview+president` touche maintenant l'index GIN et répond en quelques millisecondes quel que soit la taille de la table.

## Ce que la répartition ressemblait vraiment

La bibliothèque couvrait les fonctions atomiques bas niveau. Le code personnalisé couvrait les lacunes: un type DBAL `tsvector` que la bibliothèque ne fournissait pas, des wrappers DQL pratiques combinant `@@` et `websearch_to_tsquery` en un seul appel, et la colle applicative reliant tout ça au système d'événements de Doctrine et à API Platform. Aucune requête native n'a été nécessaire.

La répartition vaut d'être notée en général: `postgresql-for-doctrine` donne les briques atomiques PostgreSQL, mais il faut quand même les composer en quelque chose que le reste du code peut utiliser sans y penser. Le pattern `FunctionNode` et le hook `convertToDatabaseValueSQL()` sont les deux points d'extension qui rendent cette composition propre. Les deux valent d'être connus, quelle que soit la bibliothèque de départ.
