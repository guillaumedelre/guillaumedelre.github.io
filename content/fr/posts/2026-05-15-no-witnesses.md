---
title: "Aucun témoin"
date: 2026-05-15T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 3
categories: [développement]
tags: [symfony, cloud, monolog, kubernetes, 12factor, observabilité]
description: "Un service s'est crashé en production sans laisser de logs. Pourquoi fingers_crossed et les déploiements cloud ne font pas bon ménage."
---

Le service s'était crashé. On avait l'alerte. On avait le timestamp à la seconde. On avait <a href="https://grafana.com/oss/loki/" target="_blank" rel="noopener noreferrer">Loki</a> ouvert avec une requête prête.

Ce qu'on n'avait pas, c'était les logs des cinq minutes précédant le crash.

Promtail tournait. Il était healthy. Il collectait les logs de tous les autres services sans problème. Mais pour celui-ci, dans la fenêtre qui comptait, il n'y avait rien. Le service s'était crashé sans laisser de trace.

## Le setup qui semblait correct

La stack de logging était raisonnable. Chaque service écrivait du JSON structuré vers stdout avec le formatter logstash de Monolog :

```yaml
stdout:
    type: stream
    path: "php://stdout"
    level: "%env(MONOLOG_LEVEL__DEFAULT)%"
    formatter: 'monolog.formatter.logstash'
```

<a href="https://grafana.com/docs/loki/latest/" target="_blank" rel="noopener noreferrer">Promtail</a> collectait la sortie des containers via la socket Docker, parsait le JSON, extrayait des labels, poussait vers Loki :

```yaml
scrape_configs:
    -
        job_name: docker
        docker_sd_configs:
            -
                host: unix:///var/run/docker.sock
                refresh_interval: 5s
        pipeline_stages:
            -
                drop:
                    older_than: 168h
            -
                json:
                    expressions:
                        level: level
                        msg: message
                        service: service
            -
                labels:
                    level:
                    service:
        relabel_configs:
            -
                source_labels: [ '__meta_docker_container_log_stream' ]
                target_label: stream
```

Deux stages font plus de travail que les autres. Le stage `json` extrait `level` et `service` de chaque ligne de log ; le stage `labels` qui suit immédiatement les promeut en labels d'index Loki, ce qui fait de `{service="content", level="error"}` une lookup directe plutôt qu'un scan plein texte sur les lignes stockées. Le relabeling `stream` conserve si une ligne venait de stdout ou stderr — une distinction requêtable dès que Monolog envoie les erreurs vers stderr et le reste vers stdout. Le stage `drop older_than: 168h` est une soupape de sécurité : si Promtail redémarre après une longue interruption et rejoue des lignes bufferisées, tout ce qui est plus vieux de sept jours est écarté avant d'atteindre Loki.

En théorie : les logs vont vers stdout, Promtail lit stdout, les logs apparaissent dans Loki. La <a href="https://12factor.net/logs" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor</a> décrit exactement ce modèle pour le Facteur XI — traiter les logs comme des flux d'événements, écrire vers stdout, laisser l'environnement gérer la collecte et le routage.

L'application avait stdout. Promtail lisait stdout. Qu'est-ce qui pouvait mal tourner.

## Ce que fingers_crossed emporte avec lui

En production, le bloc `when@prod` remplaçait le simple handler `stream` par quelque chose de plus sophistiqué :

```yaml
when@prod:
    monolog:
        handlers:
            main:
                type: fingers_crossed
                action_level: error
                handler: main_group
                excluded_http_codes: [404]
```

La ligne `excluded_http_codes: [404]` est elle-même révélatrice : sans elle, chaque 404 d'un scanner ou d'un crawler déclenche un flush complet du buffer, déversant des mégaoctets de logs debug pour des URLs malformées. Quelqu'un avait déjà appris ça à ses dépens.

`fingers_crossed` est un pattern Monolog bien connu. L'idée est élégante : ne pas noyer les logs de production dans le bruit debug, mais si quelque chose tourne mal, retrouver rétrospectivement ce qui s'est passé avant l'erreur. Le handler bufferise chaque entrée de log en mémoire. Au moment où il voit une `error`, il flush le buffer entier vers le handler imbriqué — en donnant le contexte complet qui a précédé la défaillance.

Le problème, c'est ce qui se passe quand la défaillance n'est pas une erreur loguée. C'est un OOM kill. Un SIGKILL de l'orchestrateur. Un segfault. Un process qui arrête de répondre et est tué de force.

Dans ces cas, `fingers_crossed` n'atteint jamais son `action_level`. Le buffer existe, plein des cinq dernières minutes d'activité, et il disparaît avec le process. Les logs étaient là. Ils étaient en mémoire. Ils sont morts avant d'atteindre stdout.

Le Facteur IX du twelve-factor parle de disposabilité : les processus doivent démarrer vite et s'arrêter proprement. Sur un arrêt normal (SIGTERM), un processus bien élevé finit son travail en cours et quitte. Mais les crashes ne sont pas des arrêts propres, et les buffers mémoire ne sont pas résistants aux crashes. Le service était disposable au sens où on pouvait le redémarrer ; il ne l'était pas au sens où sa sortie était transparente.

## Les fichiers que personne ne lisait

Il y avait un deuxième problème, plus silencieux mais tout aussi persistant.

Chaque service avait un handler `main_group` qui routait les logs vers deux destinations en parallèle :

```yaml
main_group:
    type: group
    members: [main_file, stdout]

main_file:
    type: stream
    path: "%kernel.logs_dir%/%kernel.environment%.log"
    formatter: "monolog.formatter.logstash"
```

`var/log/prod.log` était écrit sur chaque service, dans chaque environnement, y compris en production. Le même contenu qui allait vers stdout allait aussi vers un fichier à l'intérieur du container. Le fichier grossissait sans rotation. Le fichier n'était pas accessible à Promtail (qui lisait depuis la socket Docker, pas depuis le filesystem du container). Le fichier consommait de l'espace disque. Personne ne le lisait.

Le channel audit était pire :

```yaml
audit_file:
    type: stream
    path: "%kernel.logs_dir%/audit.log"
    formatter: 'monolog.formatter.line'

audit:
    type: group
    members: [audit_file, stderr]
    channels: ['audit']
```

Les logs d'audit allaient vers `stderr` (visible par Promtail) et vers `audit.log` (invisible à Promtail). Le format dans le fichier était une ligne brute, pas le JSON structuré qu'attendait Promtail. En pratique, la piste d'audit existait à deux endroits : l'une requêtable, l'autre enfouie dans un répertoire de container qui ne survivait que le temps du container.

## Ce que le Facteur XI demande vraiment

Le onzième facteur est direct là-dessus : une application ne doit pas se soucier du routage ou du stockage de son flux de sortie. Elle écrit vers stdout. Tout le reste est le job de l'environnement.

Ça veut dire pas de handlers de fichiers en production. Pas en backup. Pas pour les pistes d'audit. Pas "au cas où". Du moment qu'une application se met à gérer des fichiers, elle prend en charge la rotation, la rétention, l'espace disque, et l'accessibilité — rien de tout ça n'appartient à l'intérieur d'un container.

La correction pour les handlers de fichiers est directe. Dans `when@prod`, supprimer chaque handler `*_file` et chaque group qui en inclut un. Le channel audit reçoit le même traitement : stderr uniquement, JSON structuré, pas de fichier :

```yaml
when@prod:
    monolog:
        handlers:
            stdout:
                type: stream
                path: "php://stdout"
                # défaut "warning" — configurable par déploiement via variable d'env pour du debug ciblé
                level: "%env(default:default_log_level:MONOLOG_LEVEL__DEFAULT)%"
                formatter: 'monolog.formatter.logstash'

            stderr:
                type: stream
                path: "php://stderr"
                level: error
                formatter: 'monolog.formatter.logstash'

            main:
                type: group
                members: [stdout]
                channels: ['!event', '!http_client', '!doctrine', '!deprecation', '!audit']

            audit:
                type: stream
                path: "php://stderr"
                level: debug
                formatter: 'monolog.formatter.logstash'
                channels: ['audit']
```

stdout pour le channel principal. stderr pour les erreurs et l'audit. Rien d'autre. Promtail récupère les deux via la socket Docker. Le container n'écrit rien sur disque. Et les logs d'audit sont maintenant du JSON structuré, requêtable dans Loki avec tout le reste.

## La question plus dure sur fingers_crossed

Les handlers de fichiers, c'était simple. `fingers_crossed` est plus nuancé.

Le pattern résout un vrai problème : dans un service de production actif, tout logger en debug crée du bruit et des coûts. `fingers_crossed` permet de capturer le contexte sans le payer sauf si quelque chose tourne vraiment mal. C'est un compromis raisonnable quand le mode de défaillance contre lequel on protège est une erreur applicative (une exception, une 500, une requête lente).

Ce n'est pas un compromis raisonnable quand le mode de défaillance est un crash de process. Et dans un environnement Kubernetes, les crashes de process arrivent : évictions OOM, échecs de liveness probe, pression sur les nodes. Exactement les cas où on a le plus besoin des logs.

Une approche : garder `fingers_crossed` mais réduire la taille du buffer. Par défaut il garde tout depuis le dernier reset. Mettre `buffer_size: 50` plafonne l'usage mémoire, ce qui limite aussi ce qui se perd lors d'un crash. On n'aura pas le contexte complet, mais on aura les cinquante dernières entrées. Cette voie réduit le périmètre de perte sans supprimer la cause : l'opacité dépend toujours d'un seuil d'erreur qui peut ne jamais se déclencher.

Une autre approche : accepter que les logs debug soient coûteux et monter le niveau par défaut en production. Alors on n'a plus besoin de `fingers_crossed` du tout — si info et au-dessus vont directement vers stdout, rien n'est jamais bufferisé.

L'approche retenue : supprimer `fingers_crossed`, monter le niveau par défaut à `warning`, garder un override debug disponible via variable d'env pour les investigations ciblées. Les logs qui comptent apparaissent immédiatement. Ceux qui ne comptent pas ne sont jamais écrits. Rien n'est bufferisé.

## Les crashes ne flushent pas

Le Facteur XI et le Facteur IX se rejoignent au même point : un process qui meurt en plein milieu d'une requête. [un autre article de cette série](/2026/05/16/the-cache-that-was-lying-to-us/) décrivait l'illusion d'un service qui fonctionnait parfaitement sur un pod mais se comportait silencieusement mal sur deux. C'est la même illusion, un niveau au-dessus : un service qui semblait logger correctement, jusqu'au moment où il en avait le plus besoin.

La règle pour Monolog en production est sans appel : si ça n'atteint pas stdout ou stderr avant que le process quitte, ça n'existe pas. Un handler de fichier à l'intérieur d'un container est invisible pour le collecteur de logs et meurt avec le pod. Un buffer `fingers_crossed` est invisible pour le collecteur de logs et meurt avec le process.

La production tend à créer les conditions où on a le plus besoin des logs — pression OOM, défaillances en cascade, mauvais déploiements — et c'est exactement les conditions où ces deux patterns échouent simultanément. Écrire vers stdout, adopter un niveau par défaut qui ne nécessite pas de bufferisation, et rendre l'override disponible pour quand on en a vraiment besoin. Les logs seront là. Ils n'attendront pas un seuil d'erreur qui ne se déclenche jamais.
