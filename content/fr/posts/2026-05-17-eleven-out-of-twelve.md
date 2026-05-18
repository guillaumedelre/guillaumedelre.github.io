---
title: "Onze sur douze"
date: 2026-05-17T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 8
categories: [développement]
tags: [symfony, cloud, docker, kubernetes, 12factor, doctrine, migrations]
description: "Onze facteurs résolus proprement. Le douzième : des migrations Doctrine dans l'entrypoint, en attente d'une question de gouvernance que le code seul ne peut pas trancher."
---

Le `composer.json` de chaque service avait ça dans sa section `post-install-cmd` :

```json
"post-install-cmd": [
    "bin/console cache:clear --env=prod",
    "bin/console doctrine:migrations:migrate --no-interaction"
]
```

`post-install-cmd` s'exécute pendant `composer install`, qui dans le Dockerfile de production tourne au moment du build de l'image. Il n'y a pas de base de données disponible pendant un build Docker. La commande de migration échouait silencieusement, se connectait à rien, ou était ignorée par Doctrine faute de schéma à comparer. Dans tous les cas, elle ne migrait rien.

C'est une violation nette du [Facteur XII](https://12factor.net/admin-processes) : les processus d'administration — migrations, scripts ponctuels, commandes console — doivent s'exécuter dans le même environnement que l'application, contre les vraies données de production. Les faire tourner au build inverse la relation. L'image ne devrait pas savoir qu'il y a une base de données. La base devrait être là quand l'image en a besoin.

## Le déplacement vers l'entrypoint

La commande de migration a quitté `composer.json` pour `docker-entrypoint.sh`. Le changement semble petit dans un diff. Les implications ne le sont pas.

L'entrypoint s'exécute quand le container démarre, pas quand l'image est construite. La base de données est accessible. L'entrypoint l'attend — jusqu'à 60 secondes, une tentative par seconde — avant de faire quoi que ce soit :

```sh
ATTEMPTS_LEFT_TO_REACH_DATABASE=60
until [ $ATTEMPTS_LEFT_TO_REACH_DATABASE -eq 0 ] || \
  DATABASE_ERROR=$(php bin/console dbal:run-sql -q "SELECT 1" 2>&1); do
    sleep 1
    ATTEMPTS_LEFT_TO_REACH_DATABASE=$((ATTEMPTS_LEFT_TO_REACH_DATABASE - 1))
done

if [ $ATTEMPTS_LEFT_TO_REACH_DATABASE -eq 0 ]; then
    echo "$DATABASE_ERROR"
    exit 1
fi
```

Si la base ne répond pas dans les 60 secondes, le container sort en erreur et Kubernetes le redémarre. Une fois la base prête, la migration tourne :

```sh
if [ "$( find ./migrations -iname '*.php' -print -quit )" ]; then
    php bin/console doctrine:migrations:migrate --no-interaction --all-or-nothing
fi
```

Deux changements par rapport à la commande d'origine : `--all-or-nothing` garantit que si une migration dans un batch échoue, le batch entier est annulé. Et le guard `find` passe la commande si aucun fichier de migration n'existe — utile pour les services qui n'utilisent pas les migrations Doctrine.

C'est franchement mieux. La base est là. La migration tourne dans le vrai environnement. Le flag `--all-or-nothing` apporte une atomicité que la version au build n'avait jamais eue.

## Ce que ça ne résout pas

Deux pods qui redéploient simultanément exécutent tous les deux l'entrypoint. Tous les deux atteignent la base. Tous les deux trouvent des migrations en attente. Tous les deux appellent `doctrine:migrations:migrate`.

Doctrine a un mécanisme de verrouillage : une table `doctrine_migration_versions` qui enregistre quelles migrations ont tourné, et la commande la consulte avant d'appliquer quoi que ce soit. Dans les conditions normales c'est suffisant : le deuxième pod trouve la table à jour et sort proprement. Les cas de défaillance réels sont plus précis : une migration assez longue pour dépasser le timeout du verrou de base de données, ce qui laisse un deuxième runner démarrer la même migration avant que le premier ait terminé ; ou un pod qui se vautre à mi-migration avant d'avoir enregistré la version dans la table, laissant le schéma dans un état appliqué-mais-non-enregistré que le pod suivant va tenter d'appliquer à nouveau.

La position de l'équipe est explicite : un downtime léger au déploiement est acceptable. Les versions d'application ne sont pas nécessairement compatibles avec des versions de schéma plus anciennes, donc faire tourner N et N+1 simultanément contre la même base ne serait de toute façon pas sûr. La stratégie de déploiement est Recreate : tous les anciens pods sont terminés avant que les nouveaux ne démarrent. La migration tourne au premier démarrage, sans chevauchement entre les versions. Ça fonctionne.

Mais "ça fonctionne" et "c'est la bonne architecture" sont deux réponses différentes.

## Ce que feraient les alternatives

Le [Facteur XII](https://12factor.net/admin-processes) dit que les processus d'administration doivent tourner dans des "processus ponctuels". Un processus qui tourne une fois, dans un but précis, contre l'environnement de production. L'entrypoint n'est pas ponctuel — il tourne à chaque démarrage de container, y compris les redémarrages, les événements de scaling, et les déplacements de pods par Kubernetes.

Trois alternatives existent, chacune avec une réponse différente à la question de la propriété :

**Un init container Kubernetes** tourne avant le container principal, dans le même pod. Il peut exécuter la migration, sortir, et laisser le container principal démarrer seulement après son succès. La migration est isolée du runtime applicatif. Le problème : l'init container est une image supplémentaire à construire et maintenir, et il tourne à chaque démarrage de pod — une plateforme de 14 services démarrant simultanément a toujours une race potentielle.

**Un Kubernetes Job** tourne une fois, à la demande ou déclenché par le pipeline de déploiement. Il peut être configuré pour s'exécuter avant la mise à jour des pods — séquentiel, isolé, avec un signal clair de succès ou d'échec. La race condition disparaît. La complexité se déplace vers le processus de déploiement : le Job doit se terminer avant que le rollout du Deployment commence, et le pipeline CI doit coordonner les deux.

**Un hook Helm** est le même concept exprimé de façon déclarative dans le chart Helm. Un hook `pre-upgrade` exécute la migration avant la mise à jour des pods applicatifs. C'est la réponse la plus idiomatique pour Kubernetes. Ça signifie aussi que le chart Helm est désormais responsable de l'exécution des migrations — une décision qui appartient à qui possède le chart.

Cette dernière phrase explique pourquoi l'entrypoint n'a pas changé. Déplacer les migrations hors de l'application signifie décider que l'infrastructure de déploiement — pas l'application elle-même — est responsable du schéma. C'est une question de gouvernance autant que de technique, et les questions de gouvernance prennent plus de temps à résoudre que les changements de code.

## La fin honnête

Le bloc de migration dans l'entrypoint, c'est deux lignes. Littéralement : le guard `if [ "$( find ./migrations... )" ]`, et le `php bin/console doctrine:migrations:migrate` qui suit. Onze autres facteurs ont des résolutions nettes. Le cache est passé sur Redis. Les logs vont vers stdout. Le système de fichiers est un bucket S3. Le CI assemble les images de production depuis le même commit qu'il teste. Les secrets ne voyagent plus dans les layers d'image.

Le Facteur XII a une réponse. Ce n'est juste pas la réponse finale.

Les migrations tournent au démarrage, avec une vraie base de données, avec atomicité, avec une fenêtre de retry bornée. C'est mieux que de tourner au build contre rien. La question de savoir si elles finiront dans un Job ou un hook Helm est une conversation sur qui possède le schéma — une question à laquelle un `kubectl apply` ne peut pas répondre.
