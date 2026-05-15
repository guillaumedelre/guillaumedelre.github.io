---
layout: post
title: "Revision pruning with window functions and logarithms, when DQL wasn't enough"
date: 2020-09-27
categories: [development]
tags: [doctrine, postgresql, php, symfony]
description: "How a logarithmic score and ROW_NUMBER() OVER PARTITION BY solved runaway revision table growth after DQL hit its limits."
---

Every content update on the platform creates a revision. That's by design: editors need a history they can roll back to, and the platform needs an audit trail. What nobody anticipated was the rate. Some articles go through forty saves in a single afternoon. A high-traffic piece accumulates hundreds of revisions over its lifetime. After a few months, the revision table had several million rows.

Deleting them naively wasn't an option. "Keep the last 50" loses all historical context for articles that haven't been touched in a year. "Keep one per day" loses all the detail for content that's actively being edited. What we needed was a distribution that matched how revisions are actually used: dense coverage for recent history, sparse coverage for old history.

That's a logarithmic distribution. And building it required raw SQL.

## Why simple strategies fail

The appeal of a fixed window is obvious: keep the N most recent revisions and delete the rest. It's one line of SQL and zero math. The problem is that it treats a revision from yesterday and a revision from three years ago as equally valuable, which they aren't. An editor who opens an article from 2017 doesn't need its last 50 versions; they might need one per quarter. An article that shipped this morning might need every save from the past hour.

A time-based strategy (one revision per calendar day) has the opposite problem: it's too aggressive for active content. If an article gets 30 saves between 09:00 and 10:00, all of them except one disappear. That's not history, that's erasure.

Neither strategy can express "keep more detail for recent content, less for old content." That relationship is logarithmic.

## <img src="https://cdn.simpleicons.org/postgresql" width="20" style="vertical-align: middle; margin-right: 6px;" />The scoring idea

The algorithm assigns each revision a score based on its age, then keeps only one revision per score bucket. The score formula produces high, widely-spaced values for recent revisions and small, clustered values for old ones.

The core expression, simplified, looks like this:

```sql
(
  ln( EXTRACT(epoch FROM (now() - created_at)) )
  /
  ( EXTRACT(epoch FROM (now() - created_at)) / 6000 )
)
* ( 1 / (EXTRACT(epoch FROM (now() - created_at)) / 60 / 1440) )
* 1000
```

Let `s` be the age in seconds. The formula is roughly `ln(s) / s * C`, where both the logarithm in the numerator and `s` in the denominator make the result decrease rapidly as `s` grows.

Cast to an integer, the effect is this: a revision saved 10 minutes ago might score 8432, one saved 11 minutes ago scores 8431. They're in different buckets. A revision from six months ago scores 2, one from eight months ago also scores 2. Same bucket. The window function then picks the most recent revision from each bucket and discards the rest.

The result is automatic: recent saves are all kept because each has a distinct score; old saves are thinned because many share the same score.

## <img src="https://cdn.simpleicons.org/doctrine" width="20" style="vertical-align: middle; margin-right: 6px;" />The DQL attempt that didn't ship

Window functions aren't part of DQL. Doctrine's query language has no syntax for `OVER`, `PARTITION BY`, or `ROW_NUMBER()`. Before going to raw SQL, the team tried to add them.

The `FunctionNode` approach works for single SQL functions, as we'd already seen with FTS. A `RowNumber` node emitting `ROW_NUMBER()` is trivial:

```php
class RowNumber extends FunctionNode
{
    public function getSql(SqlWalker $sqlWalker): string
    {
        return 'ROW_NUMBER()';
    }
}
```

The harder part is `OVER(PARTITION BY ... ORDER BY ...)`. An `Over` function node was drafted, with a custom `PartitionByClause` AST node to handle the `PARTITION BY` clause:

```php
class Over extends FunctionNode
{
    protected ?PartitionByClause $partitionByClause = null;
    protected ?OrderByClause $orderByClause = null;

    public function getSql(SqlWalker $sqlWalker): string
    {
        return 'OVER('
            .($this->partitionByClause
                ? $this->partitionByClause->dispatch($sqlWalker)
                : ($this->orderByClause
                    ? $this->orderByClause->dispatch($sqlWalker)
                    : ''))
            .')';
    }
}
```

It was never finished. The classes shipped marked `@deprecated` and "NOT TESTED YET". The issue is composability: DQL's `FunctionNode` works cleanly for functions that appear in WHERE clauses or SELECT expressions. A window function like `ROW_NUMBER() OVER (PARTITION BY ...)` is a different structure: it appears in a SELECT position, modifies the surrounding query semantics, and requires the parser to handle `PARTITION BY` as an extension to DQL's grammar. Making that robust enough to trust in production is a significant investment. Going to DBAL and writing the SQL directly took an afternoon.

## The query, layer by layer

The final implementation is three nested queries:

```sql
DELETE FROM revision
WHERE iri = ?
AND id NOT IN (
    SELECT id FROM (
        SELECT
            row_number() OVER (
                PARTITION BY num, iri
                ORDER BY num DESC, created_at DESC
            ) AS lines,
            *
        FROM (
            SELECT
                (
                    ( ln( EXTRACT(epoch FROM (now() - created_at)) )
                      / ( EXTRACT(epoch FROM (now() - created_at)) / 6000 ) )
                    * ( 1 / (EXTRACT(epoch FROM (now() - created_at)) / 60 / 1440) )
                    * 1000
                )::numeric::integer AS num,
                *
            FROM revision
            WHERE iri = ?
            ORDER BY created_at DESC
        ) AS lst
    ) AS rst
    WHERE lines = 1
);
```

**Inner query:** computes `num`, the integer score, for every revision of the given IRI. Rows are sorted by `created_at DESC` at this stage.

**Middle query:** runs `ROW_NUMBER() OVER (PARTITION BY num, iri ORDER BY num DESC, created_at DESC)`. Within each score bucket (`num`), revisions are numbered starting from 1 in descending age order. The most recent revision in each bucket gets `lines = 1`.

**Outer filter:** keeps only the `lines = 1` rows, one revision per score bucket.

**DELETE:** removes every revision for this IRI that isn't in the kept set.

The `PARTITION BY num, iri` is redundant on the IRI (the whole query is already filtered to one IRI), but makes the intent explicit and keeps the logic correct if the query is ever reused in a broader context.

The method is called from a companion query that identifies which IRIs have accumulated more than a threshold of revisions:

```php
public function getIrisWithMoreRevisionThan(int $maxRevisionsCount, int $limit = 0, ?int $retencyDay = null): array
{
    $queryBuilder = $this
        ->createQueryBuilder('revision')
        ->select('revision.iri')
        ->groupBy('revision.iri')
        ->having('COUNT(1) > :maxRevisions')
        ->orderBy('COUNT(1)', Order::Descending->value)
        ->setParameter('maxRevisions', $maxRevisionsCount);

    // ...

    return array_column($queryBuilder->getQuery()->getResult(), 'iri');
}
```

The two methods run together in a scheduled cleanup: find the IRIs over the threshold, prune each one.

## <img src="https://cdn.simpleicons.org/symfony" width="20" style="vertical-align: middle; margin-right: 6px;" />Wiring it to a scheduled command

The pruning query doesn't run in a request. It runs behind a Symfony command, called on a schedule.

The command takes a few options to control how aggressively it runs:

```php
#[AsCommand('app:purge:revision', 'Remove useless revisions')]
final class PurgeRevisionCommand extends Command
{
    protected function configure(): void
    {
        $this
            ->addOption('max-revisions', 'm', InputOption::VALUE_REQUIRED,
                'Revision threshold above which an IRI gets pruned', 30)
            ->addOption('limit', 'l', InputOption::VALUE_REQUIRED,
                'Max number of IRIs to process per run')
            ->addOption('delay', 'w', InputOption::VALUE_REQUIRED,
                'Delay in seconds between each IRI')
            ->addOption('retencyDay', 'r', InputOption::VALUE_OPTIONAL,
                'Only process IRIs whose last revision is older than N days');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $iris = $this->revisionRepository->getIrisWithMoreRevisionThan(
            (int) $input->getOption('max-revisions'),
            (int) $input->getOption('limit'),
            (int) $input->getOption('retencyDay'),
        );

        foreach ($iris as $iri) {
            $totalDeleted += $this->revisionRepository->deleteOldRevisionForIri($iri);
            usleep((int) $input->getOption('delay') * 1_000_000);
        }

        return Command::SUCCESS;
    }
}
```

The `--delay` option is worth noting: on a busy database, hammering a hundred `DELETE` statements back-to-back can cause lock contention. A small sleep between iterations keeps the purge from competing with production traffic.

Scheduling uses Symfony's Scheduler component rather than an external cron. The platform runs on a multi-pod cluster, and using an external cron would mean either picking one pod arbitrarily or running the purge on every pod simultaneously. The Scheduler solves that with a lock:

```php
#[AsSchedule(name: 'purge')]
final class PurgeSchedule implements ScheduleProviderInterface
{
    public function __construct(private LockFactory $lockFactory) {}

    public function getSchedule(): Schedule
    {
        $schedule = new Schedule();

        // Hourly: keep 30 revisions per IRI, process 100 IRIs per run
        $schedule->add(RecurringMessage::every('1 hour',
            new RunCommandMessage('app:purge:revision --max-revisions 30 --limit 100')
        ));

        // Nightly: for content untouched for a year, keep only 3
        $schedule->add(RecurringMessage::cron('#midnight',
            new RunCommandMessage('app:purge:revision --max-revisions 3 --limit 100 --retencyDay 365')
        ));

        // Only one pod in the cluster picks up the scheduled messages
        $schedule->lock($this->lockFactory->createLock('purge'));

        return $schedule;
    }
}
```

There are two schedules running in parallel with different thresholds. The hourly job keeps 30 revisions per IRI: a reasonable ceiling for actively-edited content. The nightly job targets only IRIs not updated in over a year, and keeps just 3. An article that hasn't moved in twelve months doesn't need thirty versions in its history.

The `schedule->lock()` call ensures only one worker picks up each scheduled message, regardless of how many pods are running. Without it, a deploy with six replicas would run six simultaneous purges at midnight.

## What it looks like in practice

An article saved 200 times will typically keep 20 to 30 revisions after pruning: most of the recent saves, a handful from last month, one or two from each quarter of the previous year. The exact count depends on the age distribution of the saves, not on an arbitrary cap.

An article last edited two years ago might end up with 5 or 6 revisions. Recent edits are all there; the old history is compressed but not gone.

It's not a perfect history. It's a useful one.

## The line between DQL and raw SQL

The window function attempt isn't a failure worth hiding. It's a useful data point: `FunctionNode` works well for scalar functions in WHERE and SELECT positions, but composing a full `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` expression in DQL is harder than it looks. The grammar extension, the AST nodes, the SQL walker integration: it's a non-trivial amount of code for something that native SQL handles in three lines.

The practical boundary is roughly this: if a PostgreSQL feature maps to a function call with fixed arity, custom DQL works. If it requires new clause syntax (window frames, CTEs, lateral joins), native DBAL is usually the better trade-off.
