---
title: "Swarrot vs Symfony Messenger : une comparaison en conditions réelles"
date: 2022-01-26
categories: [développement]
tags: [symfony, messenger, rabbitmq, amqp, comparatif]
description: "Swarrot et Symfony Messenger gèrent tous deux RabbitMQ en PHP. Voici pourquoi on a gardé Swarrot après avoir sérieusement évalué une migration."
---

On a migré une plateforme de microservices médias vers Symfony 6 début 2022. Douze services, la plupart consommant des messages depuis RabbitMQ via <a href="https://github.com/swarrot/swarrot" target="_blank" rel="noopener noreferrer">Swarrot</a>. Symfony 6 a rendu <a href="https://symfony.com/doc/current/messenger.html" target="_blank" rel="noopener noreferrer">Messenger</a> plus central que jamais, et pendant la planification de la migration un développeur a posé la question évidente : pourquoi ne pas migrer en même temps ?

Ça vient avec le framework. Ça a de la logique de retry, du support AMQP natif, de la documentation first-party. Notre setup ressemblait à de l'artisanat par comparaison.

Question légitime. On l'a prise au sérieux. Voilà ce qu'on a trouvé.

## Câbler la topologie à la main

Swarrot est une bibliothèque consumer qui enveloppe l'extension PECL AMQP. Elle lit des octets depuis une queue, les fait passer à travers une chaîne de processors (leur terme pour middleware), et laisse votre code décider quoi faire avec le payload. C'est vraiment tout.

La chaîne de middleware est la partie intéressante. Les processors sont des décorateurs imbriqués, chacun enveloppant le suivant. Les couches extérieures gèrent les préoccupations d'infrastructure avant même que le message n'atteigne la logique métier :

```yaml
middleware_stack:
    - configurator: 'swarrot.processor.signal_handler'
    - configurator: 'swarrot.processor.max_execution_time'
    - configurator: 'swarrot.processor.exception_catcher'
    - configurator: 'swarrot.processor.doctrine_object_manager'
    - configurator: 'swarrot.processor.ack'
    - configurator: 'app.processor.retry'
```

`signal_handler` est en tête parce qu'il doit intercepter `SIGTERM` avant que tout autre processor ne le voie. `ack` est près du bas parce qu'on n'acquitte le message qu'après que le traitement réussit. L'ordre n'est pas arbitraire, et il est entièrement visible dans la configuration.

La topologie est tout aussi explicite. On déclare tout soi-même : exchanges, routing keys, queues de retry, queues de lettres mortes :

```yaml
messages_types:
    content.ingest:
        exchange: e.app.content
        routing_key: q.app.content.ingest
    content.ingest_retry:
        exchange: e.app.content
        routing_key: q.app.content.ingest.retry
    content.ingest_dead:
        exchange: e.app.content
        routing_key: q.app.content.ingest.dead
```

Trois entrées par type de message logique : queue principale, queue de retry, queue de lettres mortes. Tout ce qui existe sur le broker est nommé ici. La config est verbeuse mais honnête : pas d'inférence, pas de convention plutôt que configuration. Si une queue existe dans RabbitMQ, on peut la tracer jusqu'à une seule ligne de YAML.

## Quand le nom de classe devient la route

<a href="https://symfony.com/doc/current/messenger.html" target="_blank" rel="noopener noreferrer">Symfony Messenger</a> opère un niveau plus haut. On définit une classe de message, un handler, et un transport. La bibliothèque gère la sérialisation, le routing, le retry et les queues d'échec automatiquement.

```php
class IngestContent
{
    public function __construct(
        public readonly string $contentId,
        public readonly string $source,
    ) {}
}
```

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
        routing:
            'App\Message\IngestContent': async
```

Messenger sérialise l'objet, le met sur le transport, et le désérialise de l'autre côté dans la classe correcte. Pas de topologie manuelle, pas de noms d'exchange explicites. Le nom de classe est la primitive de routing.

Cette dernière phrase est exactement là où les choses se sont compliquées pour nous.

## Quand le typage devient du couplage

Messenger suppose que le producteur et le consumer partagent une définition de classe PHP. C'est bien pour une seule application, ou pour des services qui partagent un package de contrats dédié. Dans un monorepo d'applications Symfony indépendantes, ça crée un couplage qui n'existe tout simplement pas aujourd'hui.

Prenez un message d'ingestion de contenu que douze services consomment. Avec Swarrot, chaque service lit le payload JSON brut et prend les champs qui l'intéressent. Ajouter un nouveau champ signifie mettre à jour le producteur. Les consumers qui n'ont pas besoin du champ continuent de fonctionner sans modification.

Avec Messenger, `IngestContent` doit être définie quelque part que les douze services peuvent référencer. Ça signifie soit :

- Un package PHP partagé, versionné, déployé et maintenu à travers les services. Chaque changement de schéma devient un exercice de coordination inter-services.
- Des classes dupliquées dans chaque service, qui divergent silencieusement sous la pression.

Ni l'une ni l'autre n'est gratuite. L'approche package partagé inverse le modèle de propriété : le schéma de message devient une dépendance plutôt qu'un contrat défini à la frontière. L'approche duplication est juste le problème original différé.

La différence fondamentale est ce que représente un message. Messenger est conçu pour des **commandes typées** : un objet qui porte du sens et se distribue à un handler spécifique. Swarrot traite les messages comme des **données opaques** : des octets qui coulent à travers une topologie, traités par le consumer qui écoute. Si vos messages sont des données, l'abstraction supplémentaire qu'ajoute Messenger ne vous aide pas. Elle crée de la friction.

## Le bloquant

Le problème de sérialisation était le décisif. Dans un monorepo où les services sont autonomes, partager des classes PHP entre eux n'est pas architecturalement neutre : c'est une décision de couplage qui rend les changements futurs plus difficiles. On aurait échangé une bibliothèque nominalement "legacy" pour une plus moderne tout en introduisant exactement le genre de couplage fort qu'on avait passé des années à éviter.

Il y avait des préoccupations secondaires aussi. L'extension PECL AMQP donne un accès direct aux fonctionnalités du broker (priorités de messages, TTL par queue, routing par headers exchange) que Messenger abstrait. Et migrer quinze consumers sans jour J signifie faire tourner les deux bibliothèques en parallèle, ce qui est une vraie contrainte opérationnelle.

Mais le problème de sérialisation seul aurait suffi.

## Données ou commandes : voilà la question

Le choix ne concerne pas la qualité des bibliothèques. Messenger est bien maintenu, bien documenté, et s'intègre proprement dans l'écosystème Symfony.

La question à se poser en premier est : que sont vos messages ?

Si ce sont des commandes typées avec un schéma connu et un seul consumer faisant autorité, Messenger est un choix naturel. On écrit une classe, un handler, on configure un transport, et l'infrastructure gère le reste.

Si ce sont des payloads de données consommés par plusieurs services indépendants, chacun possédant sa propre désérialisation, l'abstraction qu'ajoute Messenger joue contre vous. La topologie explicite de Swarrot et son modèle de payload brut donnent plus de contrôle là où on en a vraiment besoin.

Une vraie limitation à garder à l'esprit : Swarrot est lié à l'extension PECL AMQP, qui n'implémente qu'AMQP 0-9-1. Ce qui signifie que RabbitMQ (ou un broker compatible) est une dépendance dure. Si l'infrastructure migre un jour vers un broker AMQP 1.0 (Azure Service Bus, ActiveMQ Artemis), Swarrot ne peut pas suivre. La couche transport de Messenger abstrait ça proprement : changer de broker signifie changer un DSN, pas réécrire les consumers.

Si la portabilité de broker est une exigence, ou susceptible de le devenir, ça change significativement le calcul.

Swarrot n'est pas du legacy à migrer. Pour l'instant, c'est le bon choix : le routing AMQP comme primitive, les messages comme données, RabbitMQ comme choix d'infrastructure long terme.

Ça pourrait changer. Un package de contrats partagé, une nouvelle exigence de broker, un service greenfield qui ne porte pas le poids de la topologie existante : n'importe lequel de ces éléments pourrait faire pencher la balance vers Messenger. La bibliothèque n'est pas inadaptée à cette plateforme. Elle est peut-être juste la bonne réponse pour une version future de celle-ci.
