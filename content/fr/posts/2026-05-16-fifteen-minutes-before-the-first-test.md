---
title: "Quinze minutes avant le premier test"
date: 2026-05-16T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 5
categories: [développement]
tags: [symfony, cloud, ci, docker, kubernetes, 12factor, gitlab, testing]
description: "Comment un pipeline CI qui provisionnait une VM Azure par run — sans RabbitMQ, MinIO ni Varnish — est devenu un pipeline qui assemble l'environnement de production depuis les mêmes images qu'il livre."
---

Le pipeline avait deux stages qui n'avaient rien à voir avec le code : `provision` et `deprovision`. Entre eux, dans l'ordre : `phpunit`, `phpmetrics`, `behat`.

```yaml
stages:
  - build
  - provision
  - phpunit
  - phpmetrics
  - behat
  - deprovision
  - deploy
```

Avant que la première assertion s'exécute, quinze minutes s'étaient écoulées. Terraform avait cloné un dépôt d'infrastructure, s'était authentifié sur Azure, avait appliqué une configuration de VM. Ansible s'était connecté à la nouvelle VM, avait installé PHP, configuré l'application, câblé une base de données et une instance Redis. Ensuite les tests tournaient. Ensuite Terraform détruisait ce qu'Ansible avait construit.

Pour chaque pipeline. Depuis chaque branche. Pour chaque pull request, de l'ouverture au merge.

## Ce que ces quinze minutes ne contenaient pas

Le stage `provision` mettait en place deux services : PostgreSQL et Redis. Trois services dont l'application dépendait en production étaient absents : RabbitMQ, MinIO et Varnish.

RabbitMQ traitait tout le travail asynchrone — 56 consumers sur 14 microservices. MinIO gérait le stockage de médias. Varnish était devant le cache HTTP. En CI, aucun d'eux n'existait. Les tests qui couvraient les files de messages ou le stockage de fichiers avaient deux options : ignorer ces chemins, ou les laisser non testés jusqu'au staging. Varnish est un cas à part : les tests tapent directement dans l'application et contournent intentionnellement la couche de cache, son absence en CI est donc un choix délibéré plutôt qu'un manque.

C'est le problème que le [Facteur X](https://12factor.net/dev-prod-parity) décrit comme l'écart d'environnement. L'écart ici n'était pas une question de configuration — il était structurel. La VM était construite par Ansible depuis un script dans un dépôt séparé. Ce n'était pas une image de container. Elle n'était pas versionnée aux côtés de l'application. Si une branche modifiait la topologie de messages RabbitMQ, il n'y avait aucun moyen de tester cette modification en CI. Le changement de topologie et le code qui en dépendait ne se rencontreraient qu'en staging.

Le script de provisioning Ansible lui-même fait partie du problème :

```yaml
launch_vm:
  stage: provision
  script:
    - git clone git@gitlab.internal/infra/ci-vm.git
    - cd ci-vm
    - az login --service-principal -u $ARM_CLIENT_ID ...
    - terraform apply -var "prefix=${CI_PIPELINE_ID}-vm" ...
    - sleep 45
    - ansible-playbook behat/test-env.yml ...
```

Le `sleep 45` est là parce qu'Ansible a besoin que la VM finisse de booter avant de pouvoir s'y connecter. Ce n'est pas un oubli — c'est le délai minimum qu'une VM fraîchement provisionnée nécessite avant que SSH fonctionne. C'est inscrit dans le processus.

## Ce qui l'a remplacé

Le nouveau pipeline n'a pas de stage `provision`. Il n'a pas de stage `deprovision`. L'environnement, ce sont les images, et les images existent avant que les tests commencent.

Chaque job de test déclare ses dépendances comme des services Docker :

```yaml
services:
  - name: $REGISTRY_URL/platform/rabbitmq:$CI_COMMIT_REF_SLUG
    alias: rabbitmq
  - name: $REGISTRY_URL/platform/minio:$CI_COMMIT_REF_SLUG
    alias: minio
  - name: redis:7.4.1
    alias: redis
  - name: $ARTIFACTORY_URL/postgresql:13
    alias: postgresql
```

Les services démarrent en parallèle quand le job commence. Avant que le script de test tourne, un `before_script` attend qu'ils soient tous prêts :

```yaml
before_script:
  - $CI_PROJECT_DIR/dockerize
      -wait tcp://postgresql:5432
      -wait tcp://rabbitmq:5672
      -wait tcp://minio:9000
      -wait tcp://redis:6379
      -timeout 120s
```

Du démarrage du pipeline à la première assertion : quatre-vingt-dix secondes — en supposant que les images sont déjà dans le cache du runner ; un cold pull rallonge les choses, mais devient négligeable une fois que le pipeline a tourné une fois sur une branche donnée.

## Ce que signifie `$CI_COMMIT_REF_SLUG`

Le timing est le résultat visible. Ce qui le produit est plus intéressant encore : les noms des images.

`$REGISTRY_URL/platform/rabbitmq:$CI_COMMIT_REF_SLUG` n'est pas l'image officielle RabbitMQ de Docker Hub. C'est une image construite par le même pipeline, depuis la même branche, au même commit que le code testé. L'image RabbitMQ embarque la topologie : un `definitions.json` avec chaque exchange, chaque queue, chaque binding, chaque configuration de dead-letter — versionné dans git aux côtés de l'application qui en dépend.

Si une branche modifie la topologie de messages, le pipeline CI construit une nouvelle image RabbitMQ qui inclut ces modifications, puis exécute les tests contre elle. Le changement de topologie et le code qui en dépend sont testés ensemble, au même commit, avant que quoi que ce soit n'atteigne le staging.

La même logique s'applique à MinIO, décrite dans le [premier article de cette série](/2026/05/14/the-ghost-of-the-ci-runner/) : l'image MinIO embarque des fixtures de test préchargées. L'environnement CI n'a pas besoin d'une étape de setup pour peupler le stockage. L'état est intégré à l'artefact.

Le runner de tests lui-même suit le même pattern. Chaque job utilise une variante debug de l'image applicative — construite depuis la même branche, au même commit — avec les dépendances de test incluses :

```yaml
image: $REGISTRY_URL/platform/$service:$CI_COMMIT_REF_SLUG-debug
```

Tout l'environnement s'assemble depuis des artefacts construits au même point de l'historique git.

## Ce que ça a demandé d'abandonner

Behat et la VM provisionnée étaient couplés. La suite de tests Behat tournait contre un serveur HTTP sur la VM ; supprimer la VM signifiait supprimer Behat.

Ça s'est révélé moins bloquant que ça n'en avait l'air. La suite Behat vivait dans un dépôt séparé, nécessitait la VM pour tourner, et avait accumulé une charge de maintenance significative. PHPUnit, tournant dans le container applicatif avec les services Docker, couvrait les mêmes scénarios par un chemin plus direct : tests fonctionnels qui exercent la couche HTTP, tests unitaires pour les composants individuels, suites organisées par domaine fonctionnel et générées dynamiquement en jobs CI parallèles.

La couche BDD a disparu. La couverture de tests est restée — et pouvait désormais tourner contre les vrais services.

## Le Facteur X, appliqué

Le [Facteur X](https://12factor.net/dev-prod-parity) se lit souvent comme "utilise la même base de données en local qu'en production." C'est la version la plus simple. La version plus profonde concerne l'écart entre ce qu'on teste et ce qu'on livre.

L'écart dans l'ancien pipeline était large : une VM configurée manuellement, privée de services clés, reconstruite de zéro à chaque run. L'écart dans le nouveau pipeline est étroit : le CI assemble l'environnement depuis les mêmes images que la production, construites au même commit que le code sous test.

Les quinze minutes de Terraform et Ansible n'étaient pas seulement lentes. Elles construisaient quelque chose qui n'était pas ce que la production faisait tourner, à chaque fois, avant que le moindre test puisse commencer. Les quatre-vingt-dix secondes de `docker pull` construisent exactement ce que la production fait tourner — et les tests qui suivent testent ça, pas une approximation.
