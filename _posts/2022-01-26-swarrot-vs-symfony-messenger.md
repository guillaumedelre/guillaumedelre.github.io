---
layout: post
title: "Swarrot vs Symfony Messenger: a real-world comparison"
date: 2022-01-26
categories: [development]
tags: [symfony, messenger, architecture, rabbitmq, amqp]
description: "Swarrot and Symfony Messenger both handle RabbitMQ in PHP. Here is why we kept Swarrot after seriously evaluating a migration."
---

We migrated a media microservices platform to Symfony 6 at the start of 2022. Twelve services, most of them consuming messages from RabbitMQ via <a href="https://github.com/swarrot/swarrot" target="_blank" rel="noopener noreferrer">Swarrot</a>. Symfony 6 made <a href="https://symfony.com/doc/current/messenger.html" target="_blank" rel="noopener noreferrer">Messenger</a> more central than ever, and during the migration planning a developer asked the obvious question: why not switch at the same time?

It ships with the framework. It has retry logic, native AMQP support, first-party documentation. Our setup looked artisanal by comparison.

Fair question. We took it seriously. Here's what we found.

## :wrench: Wiring the topology by hand

Swarrot is a consumer library that wraps the PECL AMQP extension. It reads bytes from a queue, runs them through a chain of processors (their term for middleware), and lets your code decide what to do with the payload. That's really it.

The middleware chain is the interesting part. Processors are nested decorators, each wrapping the next. The outer layers handle infrastructure concerns before the message even reaches your business logic:

```yaml
middleware_stack:
    - configurator: 'swarrot.processor.signal_handler'
    - configurator: 'swarrot.processor.max_execution_time'
    - configurator: 'swarrot.processor.exception_catcher'
    - configurator: 'swarrot.processor.doctrine_object_manager'
    - configurator: 'swarrot.processor.ack'
    - configurator: 'app.processor.retry'
```

`signal_handler` sits at the top because it needs to catch `SIGTERM` before any other processor sees it. `ack` sits near the bottom because you only acknowledge the message after processing succeeds. The order is not arbitrary, and it's entirely visible in configuration.

The topology is equally explicit. You declare everything yourself: exchanges, routing keys, retry queues, dead-letter queues:

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

Three entries per logical message type: main queue, retry queue, dead-letter queue. Everything that exists on the broker is named right here. The config is verbose but honest: no inference, no convention over configuration. If a queue exists in RabbitMQ, you can trace it to a single line of YAML.

## :package: When the class name becomes the route

<a href="https://symfony.com/doc/current/messenger.html" target="_blank" rel="noopener noreferrer">Symfony Messenger</a> operates one level higher. You define a message class, a handler, and a transport. The library handles serialization, routing, retry, and failure queues automatically.

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

Messenger serializes the object, puts it on the transport, and deserializes it on the other end into the correct class. No manual topology, no explicit exchange names. The class name is the routing primitive.

That last sentence is exactly where things got complicated for us.

## :electric_plug: Where typing becomes coupling

Messenger assumes that the producer and the consumer share a PHP class definition. That's fine for a single app, or for services that share a dedicated contracts package. In a monorepo of independent Symfony applications, it creates coupling that simply doesn't exist today.

Take a content ingestion message that twelve services consume. With Swarrot, each service reads the raw JSON payload and picks the fields it cares about. Adding a new field means updating the producer. Consumers that don't need the field keep working without any modification.

With Messenger, `IngestContent` must be defined somewhere that all twelve services can reference. That means either:

- A shared PHP package, versioned, deployed, and maintained across services. Every schema change becomes a cross-service coordination exercise.
- Duplicated classes in each service, which drift silently apart under pressure.

Neither is free. The shared package approach inverts the ownership model: the message schema becomes a dependency rather than a contract defined at the boundary. The duplication approach is just the original problem deferred.

The root difference is what a message represents. Messenger is designed for **typed commands**: an object that carries meaning and dispatches to a specific handler. Swarrot treats messages as **opaque data**: bytes that flow through a topology, processed by whatever consumer happens to be listening. If your messages are data, the extra abstraction Messenger adds doesn't help you. It creates friction.

## :no_entry: The blocker

The serialization problem was the decisive one. In a monorepo where services are autonomous, sharing PHP classes between them isn't architecturally neutral: it's a coupling decision that makes future changes harder. We would have been trading a nominally "legacy" library for a more modern one while introducing exactly the kind of tight coupling we'd spent years avoiding.

There were secondary concerns too. The PECL AMQP extension gives direct access to broker features (message priorities, per-queue TTL, headers exchange routing) that Messenger abstracts away. And migrating fifteen consumers without a flag day means running both libraries in parallel, which is a real operational constraint.

But the serialization issue alone would have been enough.

## :bulb: Data or commands: that's the question

The choice isn't about library quality. Messenger is well-maintained, well-documented, and integrates cleanly into the Symfony ecosystem.

The question to ask first is: what are your messages?

If they are typed commands with a known schema and a single authoritative consumer, Messenger is a natural fit. You write a class, a handler, configure a transport, and the infrastructure handles the rest.

If they are data payloads consumed by multiple independent services, each of which owns its own deserialization, the abstraction Messenger adds works against you. Swarrot's explicit topology and raw payload model give you more control where you actually need it.

One real limitation to keep in mind: Swarrot is tied to the PECL AMQP extension, which only implements AMQP 0-9-1. That means RabbitMQ (or a compatible broker) is a hard dependency. If your infrastructure ever moves toward an AMQP 1.0 broker (Azure Service Bus, ActiveMQ Artemis), Swarrot can't follow. Messenger's transport layer abstracts this cleanly: changing brokers means changing a DSN, not rewriting consumers.

If broker portability is a requirement, or likely to become one, that changes the calculus significantly.

Swarrot isn't legacy to migrate away from. For now, it's the right fit: AMQP routing as the primitive, messages as data, RabbitMQ as a long-term infrastructure choice.

That could change. A shared contracts package, a new broker requirement, a greenfield service that doesn't carry the existing topology weight: any of these could tip the balance toward Messenger. The library isn't wrong for this platform. It may just be the right answer for a future version of it.

