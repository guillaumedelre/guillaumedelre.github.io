---
title: "Observabilité sur des conteneurs FrankenPHP avant que la migration cloud soit finie"
date: 2025-06-07
categories: [devops]
tags: [frankenphp, prometheus, grafana, caddy, observabilite]
description: "Déplacer 14 microservices PHP vers le cloud impliquait d'avoir de l'observabilité avant que la migration soit finie, pas après. La couche Caddy de FrankenPHP l'a rendu possible avec deux lignes de config."
---

Quand on fait tourner des workloads on-premise, on peut s'en sortir avec presque aucune observabilité. On a SSH. On a `top`. On a quelqu'un qui sait que le service d'authentification monte toujours le lundi matin. La connaissance institutionnelle se substitue à l'instrumentation, et personne ne budgète le temps pour la remplacer.

Puis on migre vers le cloud. La connaissance institutionnelle ne suit pas. L'accès SSH est parti ou peu pratique. Et pour la première fois, on se retrouve à fixer quatorze conteneurs FrankenPHP sans la moindre idée de ce qu'ils font réellement.

C'est le moment où on a besoin de métriques. Pas éventuellement. Avant que la migration soit terminée.

## Le problème à le faire correctement

La bonne façon d'instrumenter un service PHP pour Prometheus : ajouter une bibliothèque client, écrire des compteurs et histogrammes autour de ce qui importe, exposer une route `/metrics`, mettre à jour la config de scrape. Pour un seul service, c'est un après-midi raisonnable. Pour quatorze services en pleine migration, c'est un projet de plusieurs sprints qui entre en compétition avec tout le reste qui doit bouger.

Le calcul est gênant. On a besoin de métriques pour avoir confiance que la migration se passe bien. Mais ajouter des métriques à tout avant la migration signifie que la migration prend plus longtemps. Et plus elle prend longtemps, plus on a besoin de métriques pour savoir où on en est.

Il fallait bien que quelque chose cède.

## Ce que FrankenPHP embarque sans l'annoncer

FrankenPHP n'est pas un runtime PHP qui utilise <a href="https://caddyserver.com" target="_blank" rel="noopener noreferrer">Caddy</a> comme serveur web. La relation est inversée : Caddy est le serveur, et PHP est un module Caddy. Chaque requête HTTP passe par Caddy avant d'atteindre le code applicatif.

Caddy embarque un endpoint compatible Prometheus intégré. Pas de plugin, pas de binaire supplémentaire. Activer l'admin API et il est là.

`CADDY_GLOBAL_OPTIONS` est une variable d'environnement FrankenPHP qui injecte des directives directement dans le bloc de configuration global de Caddy. Deux lignes suffisent :

```yaml
environment:
    CADDY_GLOBAL_OPTIONS: |
        admin 0.0.0.0:2019
        metrics
```

`admin 0.0.0.0:2019` lie l'admin API à toutes les interfaces réseau — le défaut est localhost uniquement, inaccessible depuis un conteneur Prometheus sur le même réseau. `metrics` active l'endpoint.

Après ça, chaque conteneur répond à `GET :2019/metrics` avec un payload Prometheus complet. Comptes de requêtes labellisés par code de statut, histogrammes de latence, connexions actives. Aucune route ajoutée à l'application. Aucun `composer require`. Aucun changement au Dockerfile.

Une variable d'environnement, ajoutée à chaque définition de service dans un seul commit. Quatorze cibles de scrape, toutes produisant des données.

## Une image utilisable dans Grafana

La config de scrape Prometheus liste chaque service par son nom de conteneur :

```yaml
scrape_configs:
    - job_name: caddy
      metrics_path: /metrics
      static_configs:
          - targets:
              - authentication:2019
              - content:2019
              - media:2019
              # les 14 services
```

Grafana se pose au-dessus de Prometheus. Le dashboard communautaire Caddy donne les taux de requêtes, les taux d'erreur et les percentiles de latence par service, par endpoint, par code de statut. En moins d'une journée après que la migration a atterri dans le nouvel environnement, il y avait quelque chose de significatif à regarder.

La couche données suit la même logique : des exporters pour PostgreSQL, Redis et RabbitMQ scrappent au niveau infrastructure sans toucher le code applicatif. Des dashboards communautaires existent pour tous.

## Ce que cette baseline couvre réellement

Les métriques HTTP de Caddy sont des métriques de serveur web, pas des métriques applicatives. Elles répondent à : est-ce que ce service reçoit du trafic, est-ce qu'il retourne des erreurs, à quelle vitesse répond-il. Le genre de questions qu'on pose quand quelque chose est cassé et qu'on doit trier dans le noir.

Elles ne répondent pas à : combien d'éléments ont été traités aujourd'hui, quel job en arrière-plan est bloqué, quel est l'impact business de ce pic de latence. Pour ça, il faut de l'instrumentation applicative, et ce travail existe encore quand on a des choses spécifiques à mesurer.

Mais dans un contexte de migration, cette distinction compte moins qu'elle n'en a l'air. Les choses qui cassent pendant une migration cloud sont principalement des problèmes d'infrastructure : un service qui ne peut pas atteindre sa base de données, une limite mémoire définie trop bas, un consumer de queue qui a arrêté de traiter les messages. Ce sont exactement les choses que la baseline couvre.

Avoir l'instrumentation parfaite pour les événements au niveau business peut attendre que la plateforme soit stable. Avoir assez de visibilité pour savoir si la migration a réussi ne peut pas.
