---
title: "Aucun Témoin"
date: 2026-05-15T11:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 2
categories: [développement]
tags: [symfony, cloud, monolog, kubernetes, 12factor, observabilite]
description: "Un service a crashé en production sans laisser de logs derrière lui. Voici pourquoi fingers_crossed et les déploiements cloud ne font pas bon ménage."
---

Le service avait crashé. On avait l'alerte. On avait l'horodatage à la seconde près. On avait <a href="https://grafana.com/oss/loki/" target="_blank" rel="noopener noreferrer">Loki</a> ouvert et une requête prête.

Ce qu'on n'avait pas, c'étaient les logs des cinq minutes précédant le crash.

Promtail tournait. Il était healthy. Il collectait les logs de tous les autres services sans problème. Mais pour celui-là, dans la fenêtre qui comptait, il n'y avait rien. Le service avait crashé sans laisser de trace.

## La configuration qui semblait correcte

La stack de logging était raisonnable. Chaque service écrivait du JSON structuré sur stdout via le formatter logstash de Monolog :

```yaml
stdout:
    type: stream
    path: "php://stdout"
    level: "%env(MONOLOG_LEVEL__DEFAULT)%"
    formatter: 'monolog.formatter.logstash'
```

<a href="https://grafana.com/docs/loki/latest/" target="_blank" rel="noopener noreferrer">Promtail</a> collectait la sortie des containers via le socket Docker, parsait le JSON, extrayait les labels, et poussait vers Loki :

```yaml
scrape_configs:
    - job_name: docker
      docker_sd_configs:
          - host: unix:///var/run/docker.sock
      pipeline_stages:
          - json:
              expressions:
                  level: level
                  service: service
```

En théorie : les logs vont sur stdout, Promtail lit stdout, les logs apparaissent dans Loki. La <a href="https://12factor.net/logs" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor app</a> décrit exactement ce modèle pour le facteur XI — traiter les logs comme des flux d'événements, écrire sur stdout, laisser l'environnement gérer la collecte et le routage.

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

La ligne `excluded_http_codes: [404]` est elle-même un indice : sans elle, chaque 404 d'un scanner ou d'un crawler déclenche un flush complet du buffer, déversant des mégaoctets de logs debug pour des URLs malformées. Quelqu'un avait déjà appris ça à ses dépens.

`fingers_crossed` est un pattern Monolog bien connu. L'idée est élégante : ne pas noyer les logs de production dans du bruit debug, mais si quelque chose tourne mal, montrer rétrospectivement ce qui s'est passé avant l'erreur. Le handler bufferrise chaque enregistrement de log en mémoire. Dès qu'il voit un `error`, il flush tout le buffer vers le handler imbriqué — te donnant le contexte complet menant jusqu'à l'échec.

Le problème, c'est ce qui se passe quand la défaillance n'est pas une erreur loggée. C'est un OOM kill. Un SIGKILL de l'orchestrateur. Un segfault. Un processus qui cesse de répondre et est tué de force.

Dans ces cas, `fingers_crossed` n'atteint jamais son `action_level`. Le buffer existe, plein des cinq dernières minutes d'activité, et il disparaît avec le processus. Les logs étaient là. Ils étaient en mémoire. Ils sont morts avant d'atteindre stdout.

Le facteur IX de la twelve-factor app parle de jetabilité : les processus doivent démarrer vite et s'arrêter proprement. À l'arrêt propre (SIGTERM), un processus bien élevé finit son travail en cours et se termine. Mais les crashes ne sont pas des arrêts propres, et les buffers mémoire ne sont pas crash-safe. Le service était jetable dans le sens où on pouvait le redémarrer ; il ne l'était pas dans le sens où sa sortie était transparente.

## Les fichiers que personne ne lisait

Il y avait un deuxième problème, plus discret mais tout aussi persistant.

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

`var/log/prod.log` était écrit sur chaque service, dans chaque environnement, y compris la production. Le même contenu qui allait sur stdout allait aussi dans un fichier à l'intérieur du container. Le fichier grossissait sans rotation. Il n'était pas accessible à Promtail (qui lisait depuis le socket Docker, pas depuis le filesystem du container). Il consommait de l'espace disque. Personne ne le lisait.

Le canal audit était encore pire :

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

Les logs d'audit allaient sur `stderr` (visible par Promtail) et dans `audit.log` (non visible par Promtail). Le format dans le fichier était un format ligne simple, pas le JSON structuré qu'attendait Promtail. En pratique, la piste d'audit existait à deux endroits : l'un requêtable, l'autre enfoui dans un répertoire du container qui survivait seulement le temps du container.

## Ce que le facteur XI requiert vraiment

Le onzième facteur est direct à ce sujet : une app ne doit pas se préoccuper du routage ou du stockage de son flux de sortie. Elle écrit sur stdout. Tout le reste est le travail de l'environnement.

Ça veut dire : pas de handlers fichier en production. Pas comme backup. Pas pour les pistes d'audit. Pas "au cas où". Dès qu'une application commence à gérer des fichiers, elle prend en charge la rotation, la rétention, l'espace disque et l'accessibilité — rien de tout ça n'appartient à l'intérieur d'un container.

Le correctif pour les handlers fichier est simple. Dans `when@prod`, supprimer tous les handlers `*_file` et tous les groupes qui en incluent un. Le canal audit reçoit le même traitement : stderr uniquement, JSON structuré, pas de fichier :

```yaml
when@prod:
    monolog:
        handlers:
            stdout:
                type: stream
                path: "php://stdout"
                # par défaut "warning" — surchargeable par déploiement via env var pour du debug ciblé
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

stdout pour le canal principal. stderr pour les erreurs et l'audit. Rien d'autre. Promtail récupère les deux via le socket Docker. Le container n'écrit rien sur disque. Et les logs d'audit sont maintenant du JSON structuré, requêtable dans Loki avec tout le reste.

## La question plus difficile sur fingers_crossed

Les handlers fichier, c'était simple. `fingers_crossed` est plus nuancé.

Le pattern résout un vrai problème : dans un service de production chargé, logger tout au niveau debug crée du bruit et des coûts. `fingers_crossed` te permet de capturer le contexte sans en payer le prix sauf si quelque chose va vraiment mal. C'est un compromis raisonnable quand le mode de défaillance qu'on protège est une erreur au niveau application (une exception, un 500, une requête lente).

Ce n'est pas un compromis raisonnable quand le mode de défaillance est un crash de processus. Et dans un environnement Kubernetes, les crashes de processus arrivent : évictions OOM, échecs de liveness probe, pression de nœud. Exactement les cas où tu as le plus besoin des logs.

Une approche : garder `fingers_crossed` mais réduire la taille du buffer. Par défaut il garde tout depuis le dernier reset. Mettre `buffer_size: 50` plafonne l'usage mémoire, ce qui limite aussi ce qui est perdu au crash. On n'aura pas le contexte complet, mais on aura les cinquante derniers enregistrements.

Une autre approche : accepter que les logs debug soient coûteux et monter le niveau de log par défaut en production. On n'a plus besoin de `fingers_crossed` du tout — si info et au-dessus vont directement sur stdout, rien n'est jamais bufferisé.

L'approche qu'on a retenue : supprimer `fingers_crossed`, monter le niveau par défaut à `warning`, garder un override debug disponible via env var pour les investigations ciblées. Les logs qui comptent apparaissent immédiatement. Ceux qui ne comptent pas ne sont jamais écrits. Rien n'est bufferisé.

## Les crashes ne flushent pas

Le facteur XI et le facteur IX se rejoignent au même point : un processus qui meurt en pleine requête. [L'article précédent de cette série](/2026/05/15/the-cache-that-was-lying-to-us/) décrivait l'illusion d'un service qui fonctionnait parfaitement sur un pod mais se comportait mal discrètement sur deux. C'est la même illusion, une couche au-dessus : un service qui semblait logger correctement, jusqu'au moment où il en avait le plus besoin.

La règle pour Monolog en production est brutale : si ça n'atteint pas stdout ou stderr avant que le processus se termine, ça n'existe pas. Un handler fichier à l'intérieur d'un container est invisible pour le collecteur de logs et meurt avec le pod. Un buffer `fingers_crossed` est invisible pour le collecteur de logs et meurt avec le processus.

La production tend à créer les conditions où on a le plus besoin de logs — pression OOM, défaillances en cascade, mauvais déploiements — et ce sont exactement les conditions où ces deux patterns te lâchent simultanément. Écrire sur stdout, utiliser par défaut un niveau qui ne nécessite pas de bufferisation, et rendre le override disponible pour quand on a vraiment besoin de debugger quelque chose. Les logs seront là. Ils n'attendront pas un seuil d'erreur qui ne se déclenche jamais.
