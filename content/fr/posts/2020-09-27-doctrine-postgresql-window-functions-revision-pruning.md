---
title: "Élagage des révisions avec des window functions et des logarithmes, quand DQL ne suffisait plus"
date: 2020-09-27
categories: [développement]
tags: [doctrine, postgresql, php, symfony, fenetre-analytique]
description: "Comment un score logarithmique et ROW_NUMBER() OVER PARTITION BY ont résolu la croissance incontrôlable d'une table de révisions après que DQL a atteint ses limites."
---

Chaque mise à jour de contenu sur la plateforme crée une révision. C'est délibéré : les éditeurs ont besoin d'un historique sur lequel ils peuvent revenir, et la plateforme a besoin d'une piste d'audit. Ce que personne n'avait anticipé, c'était le rythme. Certains articles passent par quarante sauvegardes en un seul après-midi. Une pièce à fort trafic accumule des centaines de révisions sur sa durée de vie. Après quelques mois, la table de révisions avait plusieurs millions de lignes.

Les supprimer naïvement n'était pas une option. "Garder les 50 dernières" perd tout contexte historique pour les articles qui n'ont pas été touchés depuis un an. "Garder une par jour" perd tous les détails pour le contenu qui est activement édité. Ce dont on avait besoin, c'était une distribution qui correspondait à la façon dont les révisions sont réellement utilisées : couverture dense pour l'historique récent, couverture clairsemée pour l'ancien.

C'est une distribution logarithmique. Et la construire nécessitait du SQL brut.

## Pourquoi les stratégies simples échouent

L'attrait d'une fenêtre fixe est évident : garder les N révisions les plus récentes et supprimer le reste. C'est une ligne de SQL et zéro maths. Le problème, c'est qu'elle traite une révision d'hier et une révision d'il y a trois ans comme également précieuses, ce qu'elles ne sont pas. Un éditeur qui ouvre un article de 2017 n'a pas besoin de ses 50 dernières versions ; il pourrait avoir besoin d'une par trimestre. Un article qui a été publié ce matin pourrait avoir besoin de chaque sauvegarde de la dernière heure.

Une stratégie temporelle (une révision par jour calendaire) a le problème inverse : elle est trop agressive pour le contenu actif. Si un article reçoit 30 sauvegardes entre 09h00 et 10h00, toutes sauf une disparaissent. Ce n'est pas de l'histoire, c'est de l'effacement.

Ni l'une ni l'autre ne peut exprimer "garder plus de détails pour le contenu récent, moins pour le vieux". Cette relation est logarithmique.

## L'idée de score

L'algorithme assigne à chaque révision un score basé sur son âge, puis garde seulement une révision par bucket de score. La formule de score produit des valeurs hautes et bien espacées pour les révisions récentes, et des valeurs petites et regroupées pour les anciennes.

L'expression centrale, simplifiée, ressemble à ça :

```sql
(
  ln( EXTRACT(epoch FROM (now() - created_at)) )
  /
  ( EXTRACT(epoch FROM (now() - created_at)) / 6000 )
)
* ( 1 / (EXTRACT(epoch FROM (now() - created_at)) / 60 / 1440) )
* 1000
```

Soit `s` l'âge en secondes. La formule est grossièrement `ln(s) / s * C`, où le logarithme au numérateur et `s` au dénominateur font diminuer le résultat rapidement à mesure que `s` augmente.

Converti en entier, l'effet est le suivant : une révision sauvegardée il y a 10 minutes pourrait scorer 8432, une sauvegardée il y a 11 minutes score 8431. Elles sont dans des buckets différents. Une révision d'il y a six mois score 2, une d'il y a huit mois score aussi 2. Même bucket. La window function choisit ensuite la révision la plus récente de chaque bucket et supprime le reste.

Le résultat est automatique : les sauvegardes récentes sont toutes gardées parce que chacune a un score distinct ; les anciennes sont élagées parce que beaucoup partagent le même score.

## La tentative DQL qui n'a pas abouti

Les window functions ne font pas partie de DQL. Le langage de requête de Doctrine n'a pas de syntaxe pour `OVER`, `PARTITION BY` ou `ROW_NUMBER()`. Avant de passer au SQL brut, l'équipe a essayé de les ajouter.

L'approche `FunctionNode` fonctionne pour les fonctions SQL simples, comme on l'avait déjà vu avec la FTS. Un nœud `RowNumber` émettant `ROW_NUMBER()` est trivial :

```php
class RowNumber extends FunctionNode
{
    public function getSql(SqlWalker $sqlWalker): string
    {
        return 'ROW_NUMBER()';
    }
}
```

La partie plus difficile est `OVER(PARTITION BY ... ORDER BY ...)`. Un nœud de fonction `Over` a été ébauché, avec un nœud AST `PartitionByClause` personnalisé pour gérer la clause `PARTITION BY` :

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

Ça n'a jamais été terminé. Les classes ont été livrées marquées `@deprecated` et "NOT TESTED YET". Le problème est la composabilité : `FunctionNode` de DQL fonctionne bien pour les fonctions qui apparaissent dans les clauses WHERE ou les expressions SELECT. Une window function comme `ROW_NUMBER() OVER (PARTITION BY ...)` est une structure différente : elle apparaît dans une position SELECT, modifie la sémantique de la requête englobante, et exige que le parseur gère `PARTITION BY` comme une extension de la grammaire DQL. Rendre ça suffisamment robuste pour être fiable en production est un investissement significatif. Passer à DBAL et écrire le SQL directement a pris un après-midi.

## La requête, couche par couche

L'implémentation finale est trois requêtes imbriquées :

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

**Requête intérieure :** calcule `num`, le score entier, pour chaque révision de l'IRI donnée. Les lignes sont triées par `created_at DESC` à ce stade.

**Requête intermédiaire :** exécute `ROW_NUMBER() OVER (PARTITION BY num, iri ORDER BY num DESC, created_at DESC)`. Dans chaque bucket de score (`num`), les révisions sont numérotées à partir de 1 dans l'ordre décroissant d'âge. La révision la plus récente de chaque bucket obtient `lines = 1`.

**Filtre extérieur :** ne garde que les lignes `lines = 1`, une révision par bucket de score.

**DELETE :** supprime chaque révision pour cet IRI qui n'est pas dans l'ensemble gardé.

Le `PARTITION BY num, iri` est redondant sur l'IRI (toute la requête est déjà filtrée sur un IRI), mais rend l'intention explicite et garde la logique correcte si la requête est un jour réutilisée dans un contexte plus large.

La méthode est appelée depuis une requête complémentaire qui identifie quels IRIs ont accumulé plus qu'un seuil de révisions :

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

Les deux méthodes tournent ensemble dans un nettoyage planifié : trouver les IRIs au-dessus du seuil, élaguer chacun.

## Le câbler à une commande planifiée

La requête d'élagage ne s'exécute pas dans une requête HTTP. Elle tourne derrière une commande Symfony, appelée sur un planning.

La commande prend quelques options pour contrôler son agressivité :

```php
#[AsCommand('app:purge:revision', 'Remove useless revisions')]
final class PurgeRevisionCommand extends Command
{
    protected function configure(): void
    {
        $this
            ->addOption('max-revisions', 'm', InputOption::VALUE_REQUIRED,
                'Seuil de révisions au-dessus duquel un IRI est élaguée', 30)
            ->addOption('limit', 'l', InputOption::VALUE_REQUIRED,
                'Nombre max d\'IRIs à traiter par exécution')
            ->addOption('delay', 'w', InputOption::VALUE_REQUIRED,
                'Délai en secondes entre chaque IRI')
            ->addOption('retencyDay', 'r', InputOption::VALUE_OPTIONAL,
                'Ne traiter que les IRIs dont la dernière révision est plus vieille que N jours');
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

L'option `--delay` mérite attention : sur une base de données chargée, marteler une centaine d'instructions `DELETE` dos à dos peut provoquer de la contention de verrous. Un petit sleep entre les itérations empêche l'élagage d'entrer en concurrence avec le trafic de production.

La commande tourne derrière deux entrées crontab avec des seuils différents :

```
# Horaire : garder 30 révisions par IRI, traiter 100 IRIs par exécution
0 * * * * php bin/console app:purge:revision --max-revisions 30 --limit 100

# Nocturne : pour le contenu non touché depuis un an, garder seulement 3
0 0 * * * php bin/console app:purge:revision --max-revisions 3 --limit 100 --retencyDay 365
```

La stratégie à deux niveaux est importante. Le job horaire garde 30 révisions par IRI, ce qui est un plafond raisonnable pour le contenu activement édité. Le job nocturne cible seulement les IRIs non mis à jour depuis plus d'un an et n'en garde que 3. Un article qui n'a pas bougé depuis douze mois n'a pas besoin de trente versions dans son historique.

## Ce que ça donne en pratique

Un article sauvegardé 200 fois gardera typiquement 20 à 30 révisions après élagage : la plupart des sauvegardes récentes, quelques-unes du mois dernier, une ou deux de chaque trimestre de l'année précédente. Le décompte exact dépend de la distribution d'âge des sauvegardes, pas d'un plafond arbitraire.

Un article mis à jour pour la dernière fois il y a deux ans pourrait se retrouver avec 5 ou 6 révisions. Les modifications récentes sont toutes là ; l'ancien historique est compressé mais pas effacé.

Ce n'est pas un historique parfait. C'est un historique utile.

## La frontière entre DQL et SQL brut

La tentative window function n'est pas un échec à cacher. C'est une donnée utile : `FunctionNode` fonctionne bien pour les fonctions scalaires dans les positions WHERE et SELECT, mais composer une expression complète `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` en DQL est plus difficile qu'il n'y paraît. L'extension de grammaire, les nœuds AST, l'intégration du SQL walker : c'est une quantité non triviale de code pour quelque chose que le SQL natif gère en trois lignes.

La frontière pratique est grossièrement celle-ci : si une fonctionnalité PostgreSQL correspond à un appel de fonction d'arité fixe, le DQL personnalisé convient. Si elle nécessite une nouvelle syntaxe de clause (frames de fenêtre, CTEs, lateral joins), le DBAL natif est généralement le meilleur compromis.
