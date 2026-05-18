---
title: "Démarré ne veut pas dire prêt"
date: 2026-05-17T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 7
categories: [développement]
tags: [symfony, cloud, kubernetes, docker, 12factor]
description: "Le script d'entrypoint qui fonctionne parfaitement sous Docker Compose a cinq responsabilités. Sur Kubernetes, chacune d'elles a sa place ailleurs."
---

Le rolling deploy avait l'air propre. Un nouveau pod démarrait. Kubernetes voyait le healthcheck passer — `php -v` renvoyait zéro — et commençait à router du trafic vers le nouveau container.

Pendant les quarante secondes suivantes — sur les soixante possibles — ce container était en train de poller la base de données.

Les requêtes qui atterrissaient dessus pendant cette fenêtre récoltaient des erreurs. Pas beaucoup — la fenêtre était courte — mais assez pour apparaître comme du bruit dans le monitoring. Le genre de bruit qu'on classe comme « problème réseau transitoire » et qu'on ne signale nulle part. Le déploiement a réussi. Le pod a fini par devenir prêt. Le mécanisme qui en était la cause était toujours là, attendant le prochain déploiement.

Le script d'entrypoint fait cinq choses avant que FrankenPHP démarre : copier un fichier de version, vérifier le répertoire vendor, attendre jusqu'à soixante secondes la base de données, jouer les migrations en attente, installer les assets et configurer les permissions filesystem. Sous Docker Compose, c'est invisible. Sur Kubernetes, l'écart devient du trafic en erreur.

## L'écart entre démarré et prêt

Kubernetes décide d'envoyer du trafic à un pod en surveillant sa readiness probe. Un pod dont la readiness probe passe reçoit des requêtes. Un pod dont la readiness probe échoue est retiré de la rotation du load balancer jusqu'à ce qu'il récupère. C'est le mécanisme qui rend les rolling deploys sûrs : Kubernetes ne bascule pas vers un nouveau pod tant que ce pod n'indique pas qu'il est prêt.

Le compose.yaml définit un healthcheck sur chaque service :

```yaml
healthcheck:
    test: [ "CMD", "php", "-v" ]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 10s
```

`php -v` réussit dès que le binaire PHP est présent — ce qui est vrai depuis la première milliseconde de vie du container. Le `start_period: 10s` donne dix secondes avant que les vérifications commencent. Mais la boucle de polling de l'entrypoint tourne jusqu'à soixante secondes avant que FrankenPHP démarre. À la dixième seconde, le healthcheck passe. L'application attend toujours la base de données.

Le Dockerfile a un meilleur signal :

```dockerfile
HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
```

Le port 2019 est le serveur de métriques intégré à Caddy, embarqué directement dans FrankenPHP. L'endpoint est compatible Prometheus et ne répond qu'une fois que la stack HTTP de Caddy est pleinement initialisée et que les workers PHP acceptent des connexions. `php -v` se termine en cinquante millisecondes quel que soit l'état de l'application — il vérifie le binaire, pas le serveur. `:2019/metrics` ne répond que quand le serveur sert vraiment. Ce n'est pas non plus un endpoint ajouté exprès pour la probe : chaque service de la plateforme l'a déjà scraped par Prometheus, donc le signal est actif indépendamment de toute configuration de healthcheck.

C'est plus proche. Mais sur Kubernetes, l'instruction `HEALTHCHECK` du Dockerfile est totalement ignorée. Kubernetes utilise sa propre configuration de probes. Sans définitions de probes explicites dans les manifests Kubernetes, il n'y a aucune vérification de readiness — et un pod est considéré prêt dès que son container démarre.

Ce qui signifie : le pod démarre, l'entrypoint commence à poller, Kubernetes route du trafic, l'application n'est pas encore en état de le traiter. Les requêtes arrivent sur un container qui n'est pas prêt à les gérer.

## Trois signaux, trois questions

Kubernetes sépare le cycle de vie d'un container en trois questions distinctes, chacune avec son propre type de probe :

**startupProbe** — « L'application a-t-elle fini de démarrer ? » Se déclenche à répétition jusqu'à ce qu'elle passe, puis passe la main à la liveness. Empêche la liveness probe de tuer un container qui est légitimement long à initialiser. Pour un container dont l'entrypoint peut prendre soixante secondes, c'est l'outil adapté.

**readinessProbe** — « L'application est-elle prête à traiter des requêtes ? » Échoue et passe tout au long de la vie du container. Quand elle échoue, le pod est retiré du load balancer. C'est ce qui rend un rolling deploy sûr.

**livenessProbe** — « L'application est-elle toujours vivante ? » Si elle échoue, Kubernetes redémarre le container. Conçue pour détecter les processus bloqués, pas les démarrages lents.

La boucle de polling de soixante secondes appartient à la patience de la startupProbe, pas au code applicatif :

```yaml
startupProbe:
    httpGet:
        path: /metrics
        port: 2019
    failureThreshold: 12    # 12 tentatives × 5s = 60s max
    periodSeconds: 5
```

Une fois la startupProbe passée, une readinessProbe sur le même endpoint prend le relais — indiquant à Kubernetes quand le pod peut recevoir du trafic — et une livenessProbe surveille les processus bloqués. Mais c'est la startupProbe qui absorbe le démarrage lent. La boucle de polling de l'entrypoint devient redondante : son rôle était de maintenir le container en vie pendant que la base de données devenait disponible. Sans elle, l'application tente de se connecter, échoue, et le container quitte — Kubernetes redémarre alors le pod, et la startupProbe maintient son cycle de tentatives jusqu'à ce que la base réponde et que l'application démarre proprement. La responsabilité du retry passe de l'intérieur de l'entrypoint à l'orchestrateur, ce qui est exactement là où elle devrait être.

## Le problème des migrations

La boucle de polling est le problème le plus visible, mais les migrations en créent un plus subtil.

Avec un rolling deploy et deux replicas, Kubernetes démarre un nouveau pod pendant que l'ancien sert encore du trafic. Les deux pods jouent le même entrypoint. Les deux atteignent `doctrine:migrations:migrate`.

La table de migrations de Doctrine trace quelles migrations ont déjà été exécutées, donc une migration complétée ne se jouera pas deux fois. Mais si deux pods démarrent simultanément et voient tous les deux une migration en attente, les deux tentent de la jouer en même temps. Si c'est sûr ou non dépend de la migration : les changements de schéma additifs passent en général bien ; les destructifs moins. Et on ne choisit pas lesquels s'exécutent lors d'un déploiement qui n'a pas prévu de se coordonner. `--all-or-nothing` enveloppe les migrations dans une transaction et fait un rollback si l'une échoue — c'est une question d'atomicité au sein d'une seule exécution, pas de coordination entre processus.

L'approche plus propre sépare ces deux préoccupations en deux init containers : l'un qui attend la base de données, l'autre qui joue les migrations. Le container principal ne démarre qu'une fois les deux terminés :

```yaml
initContainers:
    - name: wait-for-db
      image: authentication:latest
      command: ["php", "bin/console", "dbal:run-sql", "-q", "SELECT 1"]
    - name: migrate
      image: authentication:latest
      command: ["php", "bin/console", "doctrine:migrations:migrate", "--no-interaction", "--all-or-nothing"]
```

Les deux init containers réutilisent la même image que l'application. Ce n'est pas du gaspillage : ils ont besoin du même binaire PHP et du même câblage d'environnement pour atteindre la base de données et trouver les classes de migration. Une image dédiée plus légère réduirait le temps de démarrage, mais nécessiterait de maintenir une installation PHP séparée en synchronisation avec l'image principale.

Même avec des init containers, plusieurs pods démarrant simultanément — déploiement initial, après une défaillance de nœud, ou sous pression d'autoscaling — tenteront chacun de jouer les migrations. Le résoudre proprement — via un hook pre-upgrade Helm, une stratégie `maxSurge: 0`, ou un Job de migration séparé — est un sujet en soi. Ce qui compte ici, c'est que l'entrypoint est le mauvais endroit pour prendre cette décision : il ne peut pas se coordonner entre pods, et il lie l'exécution des migrations au démarrage de l'application d'une façon difficile à démêler plus tard. La question de quelle alternative convient à cette codebase — et pourquoi l'entrypoint n'a pas encore été remplacé — fait l'objet de [l'article suivant dans cette série](/2026/05/17/eleven-out-of-twelve/).

Le Facteur XII de la <a href="https://12factor.net/admin-processes" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor</a> — les processus d'administration tournent dans le même environnement que l'application — est respecté dans les deux cas. La question est de savoir si « même environnement » signifie « même script d'entrypoint » ou « même image, processus séparé ». Sur Kubernetes, le second est plus sûr.

## La vraie responsabilité de l'entrypoint

Enlever l'attente de la base de données (maintenant une startupProbe ou un init container), les migrations (maintenant un init container ou un Job), et l'installation des assets (une opération de build-time qui appartient au Dockerfile), et l'entrypoint n'a plus qu'une seule responsabilité : démarrer l'application.

```sh
exec docker-php-entrypoint "$@"
```

Le Facteur IX de la twelve-factor app demande un démarrage rapide et un arrêt propre. Un container dont le démarrage prend soixante secondes parce qu'il attend des dépendances externes n'est pas rapide. Ça signifie des rolling deploys lents, une reprise après crash lente, et un scale-out horizontal qui crée une fenêtre de soixante secondes avant que chaque nouveau pod contribue.

Le démarrage rapide n'est pas juste un confort. C'est ce qui fait fonctionner le reste du modèle cloud. Quand un pod peut démarrer en secondes, l'orchestrateur peut scaler agressivement et récupérer vite. Quand ça prend une minute, on ajoute des marges partout — timeouts de probes plus longs, fenêtres de déploiement plus larges, politiques de scaling plus conservatrices — et le système devient rigide.

## La taxe Docker Compose

L'entrypoint accumule ces responsabilités pour une raison. Sous Docker Compose, il n'y a pas de concept d'init container. Pas de startupProbe. Les services déclarent `depends_on`, mais sans conditions de santé, c'est juste de l'ordre de démarrage — pas de la readiness. L'entrypoint comble le vide.

Ce n'est pas un défaut de conception. C'est une adaptation raisonnable aux contraintes de Docker Compose. Le script fonctionne. Il gère les cas limites (le timeout de la base, les erreurs irrécupérables, le répertoire de migrations absent). Quelqu'un l'a testé.

Le problème, c'est l'hypothèse que le même script fonctionne aussi bien sur Kubernetes. Il tourne. L'application finit par démarrer. Mais il contourne le système de probes qui rend les déploiements Kubernetes fiables, et il place la responsabilité des migrations à un endroit où la coordination entre pods est difficile à raisonner.

Plusieurs des changements de cette série — [stockage des médias](/2026/05/14/the-ghost-of-the-ci-runner/), [secrets dans les images](/2026/05/14/what-survives-the-build/), [handlers de logs](/2026/05/15/no-witnesses/), [dépendances de services](/2026/05/15/the-host-that-hid-the-graph/), [parité d'environnement CI](/2026/05/16/fifteen-minutes-before-the-first-test/), [adaptateurs de cache](/2026/05/16/the-cache-that-was-lying-to-us/) — étaient des changements au code applicatif ou à la configuration. Celui-ci est différent. Il demande à l'infrastructure de comprendre ce que « prêt » signifie pour cette application, et il demande à l'entrypoint de céder des responsabilités qu'il détient actuellement.

C'est une conversation plus difficile. Mais la startupProbe attend.
