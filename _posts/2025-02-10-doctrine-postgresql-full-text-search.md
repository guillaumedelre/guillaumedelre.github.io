---
layout: post
title: "PostgreSQL full-text search through Doctrine, without a line of raw SQL"
date: 2025-02-10
categories: [development]
tags: [doctrine, postgresql, symfony, php, api-platform]
description: "How we layered custom DBAL types and DQL wrappers on top of postgresql-for-doctrine to bring PostgreSQL full-text search to a Symfony API Platform project."
---

The search box on the media library returned results in 800 milliseconds on staging. Production had forty times more rows. The query plan showed a sequential scan: no index involved, no way to fix it with a standard B-tree. The product team also wanted multi-word search: type "interview president", get results containing both words. A `LIKE` query with wildcards has no clean way to express that without multiple independent conditions, each requiring its own scan.

PostgreSQL has had built-in full-text search for over fifteen years. The platform was already on PostgreSQL. The catch: the project uses Doctrine ORM, and Doctrine doesn't natively know what a `tsvector` is.

A community library, <a href="https://github.com/martin-georgiev/postgresql-for-doctrine" target="_blank" rel="noopener noreferrer">postgresql-for-doctrine</a>, covers part of that gap. It registers basic DQL functions like `TO_TSQUERY`, `TO_TSVECTOR`, and the `@@` match operator as separate atomic pieces. The foundation was there. Three things still had to be built on top.

## <img src="https://cdn.simpleicons.org/doctrine" width="20" style="vertical-align: middle; margin-right: 6px;" />The type Doctrine has never seen

<a href="https://www.postgresql.org/docs/current/datatype-textsearch.html" target="_blank" rel="noopener noreferrer">PostgreSQL's full-text search</a> is built around two types: `tsvector` (a pre-processed list of normalized tokens) and `tsquery` (a search expression). You maintain a `tsvector` column, index it with GIN, and query with the `@@` match operator.

Doctrine's DBAL ships no `tsvector` type. Declaring `#[ORM\Column(type: 'tsvector')]` without registering it first throws a `UnknownColumnTypeException`. The fix is a custom DBAL type:

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

The interesting method is `convertToDatabaseValueSQL()`. Doctrine calls it to wrap the SQL placeholder before the value reaches the database. The written value automatically becomes `to_tsvector('simple', ?)` at the DBAL boundary with no extra step needed on the calling side.

Register the type in `doctrine.yaml`, then map the column on the entity:

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

PHP-side, the value is a plain string. The conversion to a proper `tsvector` happens invisibly at the DBAL layer.

We used the `'simple'` dictionary, which tokenizes on whitespace and punctuation without language-specific stemming. The platform handles multiple languages, and French stemming rules would break Spanish. Simple is good enough for phonetics.

## Keeping the column current

A `tsvector` column is derived data: it has to stay in sync with the source fields whenever the entity changes. A Doctrine event listener handles that:

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

Before every persist and update, the subscriber concatenates the fields that should be searchable into `textSearch`. Doctrine flushes the combined string, the DBAL type wraps it in `to_tsvector('simple', ...)`, and PostgreSQL stores the tokenized form.

One subtlety: the PHP-side value is `"title caption"`, not the actual tsvector output. The database shows `'caption' 'title'` (sorted tokens), but the entity holds a plain string. That's expected: the conversion is a DBAL responsibility, not a PHP one. It can be confusing to debug until you remember where the boundary is.

## <img src="https://cdn.simpleicons.org/postgresql" width="20" style="vertical-align: middle; margin-right: 6px;" />Extending DQL with FTS operators

Doctrine's DQL covers common SQL operations, but anything PostgreSQL-specific is out of scope. That's where `postgresql-for-doctrine` starts: it registers `TO_TSQUERY`, `TO_TSVECTOR`, and `TSMATCH` as individual DQL functions. Writing a full-text query in DQL without it would mean dropping to native SQL entirely.

The library's functions are atomic, though. Each maps to one SQL call. Expressing a full match check in DQL looks like `TSMATCH(o.textSearch, TO_TSQUERY(:term))`. Readable enough, but the team wanted something more compact: a single DQL function that encodes both the match operator and the query type, including `websearch_to_tsquery`, which `postgresql-for-doctrine` didn't ship.

The solution is <a href="https://www.doctrine-project.org/projects/doctrine-orm/en/latest/cookbook/dql-user-defined-functions.html" target="_blank" rel="noopener noreferrer">custom DQL functions</a> via `FunctionNode`. You parse the DQL syntax, then emit SQL. All FTS functions share the same two-argument signature, so an abstract base class handles parsing:

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

Each concrete class implements `getSql()` to emit its PostgreSQL expression:

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

// ts_rank(e.textSearch, to_tsquery(:term)) for relevance ordering
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

`websearch_to_tsquery` is the right choice for user-facing search: spaces become AND, quoted strings become phrases, `-word` excludes a term. No need to teach users to type `interview & president`. It was added in PostgreSQL 11. On older versions, `plainto_tsquery` is the closest equivalent.

## <img src="https://cdn.simpleicons.org/symfony" width="20" style="vertical-align: middle; margin-right: 6px;" />The API Platform filter and the GIN index

With the DQL functions registered, the API Platform filter is straightforward. A custom `AbstractFilter` calls the DQL function directly in the `QueryBuilder`:

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

Apply it on the entity alongside the index declaration:

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

The `USING gin` option is non-negotiable. A standard B-tree index on a `tsvector` column is useless: PostgreSQL can't use it for `@@` queries. GIN (Generalized Inverted Index) works differently: it indexes each token individually, so lookups by any token are `O(log n)` rather than `O(n)`. Without it, you've built a fast-looking system that still does a full table scan.

A `GET /media?textSearch=interview+president` now hits the GIN index and returns in single-digit milliseconds regardless of table size.

## What the split actually looked like

The library covered the low-level atomic functions. The custom code covered the gaps: a `tsvector` DBAL type the library didn't provide, convenience DQL wrappers that combined `@@` and `websearch_to_tsquery` into a single call, and the application-specific glue connecting it all to Doctrine's event system and API Platform. Nothing needed to drop to a native query.

The split is worth noting in general: `postgresql-for-doctrine` gives you the atomic PostgreSQL building blocks, but you still need to compose them into something the rest of the codebase can use without thinking about it. The `FunctionNode` pattern and the `convertToDatabaseValueSQL()` hook are the two extension points that make that composition clean. Both are worth knowing about, regardless of what library you start from.
