---
title: "Prêt n'est pas pareil que Démarré"
date: 2026-05-15T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 6
categories: [développement]
tags: [symfony, cloud, kubernetes, docker, 12factor, healthcheck]
description: "Le script d'entrée qui fonctionne parfaitement dans Docker Compose a cinq responsabilités. Dans Kubernetes, chacune appartient ailleurs."
---

Le déploiement rolling avait l'air propre. Un nouveau pod a démarré. Kubernetes a vu le healthcheck passer — `php -v` a retourné zéro — et a commencé à router du trafic vers le nouveau conteneur.

Pendant les quarante secondes suivantes — sur un possible de soixante — ce conteneur interrogeait la base de données.

Les requêtes qui ont atterri dessus pendant cette fenêtre ont obtenu des erreurs. Pas beaucoup — la fenêtre était courte — mais assez pour apparaître comme du bruit dans le monitoring. Le genre de bruit qu'on rejette comme un problème réseau transitoire et qu'on ne consigne nulle part. Le déploiement a réussi. Le pod est finalement devenu prêt. Le mécanisme qui l'avait causé était toujours là, attendant le prochain déploiement.

Le script d'entrée fait cinq choses avant que FrankenPHP démarre : copier un fichier de version, vérifier le répertoire vendor, attendre jusqu'à soixante secondes que la base de données soit disponible, lancer les migrations en attente, installer les assets et définir les permissions du filesystem. Dans Docker Compose, c'est invisible. Dans Kubernetes, l'écart devient du trafic.

## L'écart entre démarré et prêt

Kubernetes décide s'il faut envoyer du trafic à un pod en regardant sa readiness probe. Un pod dont la readiness probe passe reçoit des requêtes. Un pod dont la readiness probe échoue est retiré de la rotation du load balancer jusqu'à ce qu'il récupère. C'est le mécanisme qui rend les déploiements rolling sûrs : Kubernetes ne bascule pas vers un nouveau pod jusqu'à ce que ce pod dise qu'il est prêt.

Le compose.yaml définit un healthcheck sur chaque service :

```yaml
healthcheck:
    test: [ "CMD", "php", "-v" ]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 10s
```

`php -v` réussit dès que le binaire PHP est présent — ce qui est vrai depuis la première milliseconde de vie du conteneur. Le `start_period: 10s` donne dix secondes avant que les vérifications commencent. Mais la boucle d'interrogation de l'entrypoint tourne jusqu'à soixante secondes avant même que FrankenPHP démarre. À la dixième seconde, le healthcheck passe. L'application attend toujours la base de données.

Le Dockerfile a un signal meilleur :

```dockerfile
HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
```

Ça vérifie l'endpoint metrics que FrankenPHP expose, avec une période de grâce de soixante secondes pour couvrir la boucle d'interrogation. C'est plus proche. Mais dans Kubernetes, l'instruction `HEALTHCHECK` est complètement ignorée. Kubernetes utilise sa propre configuration de probe. Sans définitions de probe explicites dans les manifests Kubernetes, il n'y a pas de vérifications de readiness — et un pod est considéré prêt dès que son conteneur démarre.

Ce qui signifie : le pod démarre, l'entrypoint commence à interroger, Kubernetes route du trafic, l'application n'est pas encore prête à servir. Les requêtes arrivent à un conteneur qui n'est pas prêt à les gérer.

## Trois signaux, trois questions

Kubernetes sépare le cycle de vie du conteneur en trois questions distinctes, chacune avec son propre type de probe :

**startupProbe** — "L'application a-t-elle fini de démarrer ?" Se déclenche de manière répétée jusqu'à ce qu'elle passe, puis passe la main à la liveness. Empêche la liveness probe de tuer un conteneur qui est légitimement lent à initialiser. Pour un conteneur dont l'entrypoint peut prendre soixante secondes, c'est le bon outil.

**readinessProbe** — "L'application est-elle prête à traiter des requêtes ?" Échoue et passe tout au long de la vie du conteneur. Quand elle échoue, le pod est retiré du load balancer. C'est ce qui rend un déploiement rolling sûr.

**livenessProbe** — "L'application est-elle toujours vivante ?" Si elle échoue, Kubernetes redémarre le conteneur. Conçue pour attraper les processus bloqués, pas les démarrages lents.

La boucle d'interrogation de soixante secondes appartient à la patience de la startupProbe, pas dans le code applicatif :

```yaml
startupProbe:
    httpGet:
        path: /metrics
        port: 2019
    failureThreshold: 12    # 12 tentatives × 5s = 60s max
    periodSeconds: 5
```

Une fois que la startupProbe passe, une readinessProbe sur le même endpoint prend le relais — disant à Kubernetes quand le pod est sûr à recevoir du trafic — et une livenessProbe surveille les processus bloqués. Mais la startupProbe est celle qui absorbe le démarrage lent. La boucle d'interrogation de l'entrypoint devient redondante : si la base de données n'est pas disponible dans la fenêtre de la startup probe, Kubernetes tue le conteneur et réessaie.

## Le problème des migrations

La boucle d'interrogation est le problème le plus visible, mais les migrations en créent un plus subtil.

Avec un déploiement rolling et deux replicas, Kubernetes démarre un nouveau pod pendant que l'ancien sert toujours du trafic. Les deux pods exécutent le même entrypoint. Les deux atteignent `doctrine:migrations:migrate`.

La table des migrations de Doctrine suit quelles migrations ont déjà été exécutées, donc une migration terminée ne tournera pas deux fois. Mais si deux pods démarrent simultanément et voient tous les deux une migration en attente, les deux essaient de l'exécuter en même temps. Si c'est sûr dépend de la migration : les changements de schéma additifs sont généralement bien ; les destructifs moins. Et on ne choisit pas lesquels tournent lors d'un déploiement qui ne s'attendait pas à coordonner. `--all-or-nothing` enveloppe les migrations dans une transaction et annule tout si une échoue — c'est à propos de l'atomicité au sein d'une seule exécution, pas de la coordination entre processus.

L'approche plus propre sépare les deux préoccupations en deux init containers : un qui attend la base de données, un qui lance les migrations. Le conteneur principal ne démarre qu'une fois les deux terminés :

```yaml
initContainers:
    - name: wait-for-db
      image: authentication:latest
      command: ["php", "bin/console", "dbal:run-sql", "-q", "SELECT 1"]
    - name: migrate
      image: authentication:latest
      command: ["php", "bin/console", "doctrine:migrations:migrate", "--no-interaction", "--all-or-nothing"]
```

Même avec des init containers, plusieurs pods démarrant simultanément — déploiement initial, après une défaillance de nœud, ou sous pression d'autoscaling — tenteront chacun d'exécuter des migrations. Résoudre ça correctement — via un hook pre-upgrade Helm, une stratégie `maxSurge: 0`, ou un Job de migration séparé — est un sujet en soi. Ce qui compte ici est que l'entrypoint est le mauvais endroit pour héberger cette décision : il ne peut pas coordonner entre les pods, et il lie l'exécution des migrations au démarrage de l'application d'une façon difficile à démêler plus tard.

Le Factor XII de la <a href="https://12factor.net/admin-processes" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor</a> — les processus admin tournent dans le même environnement que l'application — est satisfait dans les deux cas. La question est de savoir si "même environnement" signifie "même script d'entrypoint" ou "même image, processus séparé". Dans Kubernetes, le second est plus sûr.

## Ce que le vrai travail de l'entrypoint est

Supprimer l'attente de la base de données (maintenant une startupProbe ou un init container), les migrations (maintenant un init container ou un Job), et l'installation des assets (une opération au moment du build qui appartient au Dockerfile), et l'entrypoint a une seule responsabilité restante : démarrer l'application.

```sh
exec docker-php-entrypoint "$@"
```

Le Factor IX de la twelve-factor app demande un démarrage rapide et un arrêt gracieux. Un conteneur dont le démarrage prend soixante secondes parce qu'il attend des dépendances externes n'est pas rapide. Ça signifie que les déploiements rolling sont lents, la récupération après un crash est lente, et le scale-out horizontal crée un écart de soixante secondes avant que chaque nouveau pod contribue.

Un démarrage rapide n'est pas juste un "nice-to-have". C'est ce qui fait fonctionner le reste du modèle cloud. Quand un pod peut démarrer en secondes, l'orchestrateur peut scaler agressivement et récupérer rapidement. Quand ça prend une minute, on ajoute de la marge partout — des timeouts de probe plus longs, des fenêtres de déploiement plus grandes, des politiques de scaling plus conservatives — et le système devient rigide.

## La taxe Docker Compose

L'entrypoint accumule ces responsabilités pour une raison. Dans Docker Compose, il n'y a pas de concept d'init container. Il n'y a pas de startupProbe. Les services déclarent `depends_on`, mais sans conditions de santé, c'est juste un ordonnancement du démarrage — pas de la readiness. L'entrypoint comble le vide.

Ce n'est pas un défaut de conception. C'est une adaptation raisonnable aux contraintes de Docker Compose. Le script fonctionne. Il gère les cas limites (le timeout de la base de données, les erreurs irrécupérables, le répertoire de migrations manquant). Quelqu'un l'a testé.

Le problème est l'hypothèse que le même script fonctionne également bien dans Kubernetes. Il tourne. L'application finit par démarrer. Mais il contourne le système de probes qui rend les déploiements Kubernetes fiables, et il met la responsabilité des migrations dans un endroit où la coordination entre pods est difficile à raisonner.

La série de migrations que cette codebase a traversées — [adaptateurs de cache](/2026/05/15/the-cache-that-was-lying-to-us/), [handlers de logs](/2026/05/15/no-witnesses/), [secrets d'image](/2026/05/15/layers-remember-everything/), [coordination du scheduler](/2026/05/15/the-job-that-never-exits/), [stockage media](/2026/05/15/three-adapters-one-variable/) — toutes étaient des changements au code applicatif ou à la configuration. Celle-ci est différente. Elle nécessite que l'infrastructure acquière une compréhension de ce que "prêt" signifie pour cette application, et elle nécessite que l'entrypoint abandonne des responsabilités qu'il possède actuellement.

C'est une conversation plus difficile. Mais la startupProbe l'attend.
