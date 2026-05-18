---
title: "Fifteen Minutes Before the First Test"
date: 2026-05-16T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 5
categories: [development]
tags: [symfony, cloud, ci, docker, kubernetes, 12factor, gitlab, testing]
description: "How a CI pipeline that provisioned an Azure VM per run — missing RabbitMQ, MinIO, and Varnish — became one that assembles the production environment from the same images it ships."
---

The pipeline had two stages that had nothing to do with code: `provision` and `deprovision`. Between them, in sequence, came `phpunit`, `phpmetrics`, and `behat`.

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

Before the first assertion ran, fifteen minutes had passed. Terraform had cloned an infrastructure repository, authenticated to Azure, and applied a VM configuration. Ansible had connected to the new VM, installed PHP, configured the application, wired up a database and a Redis instance. Then the tests ran. Then Terraform destroyed what Ansible had built.

For every pipeline. From every branch. For every pull request, from open to merge.

## What those fifteen minutes were missing

The `provision` stage set up two services: PostgreSQL and Redis. Three services that the application depended on in production were absent: RabbitMQ, MinIO, and Varnish.

RabbitMQ processed all asynchronous work — 56 consumers across 14 microservices. MinIO handled media storage. Varnish fronted the HTTP cache. In CI, none of them existed. Tests that exercised message queuing or file storage had two options: skip these paths, or leave them untested until staging. Varnish is a different case: tests hit the application directly and intentionally bypass the cache layer, so its absence in CI is a deliberate choice rather than a gap.

This is the problem [Factor X](https://12factor.net/dev-prod-parity) describes as the environment gap. The gap here wasn't a matter of configuration — it was structural. The VM was built by Ansible from a script in a separate repository. It wasn't a container image. It wasn't versioned alongside the application. If a branch modified the RabbitMQ message topology, there was no way to test that modification in CI. The topology change and the code that relied on it would only meet in staging.

The Ansible provisioning script itself is part of the problem:

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

The `sleep 45` is there because Ansible needs the VM to finish booting before it can connect. It's not an oversight — it's the minimum time a freshly provisioned VM needs before SSH works. It's baked into the process.

## What replaced it

The new pipeline has no `provision` stage. It has no `deprovision` stage. The environment is the images, and the images exist before the tests begin.

Each test job declares its dependencies as Docker services:

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

The services start in parallel when the job begins. Before the test script runs, a `before_script` waits for all of them to be ready:

```yaml
before_script:
  - $CI_PROJECT_DIR/dockerize
      -wait tcp://postgresql:5432
      -wait tcp://rabbitmq:5672
      -wait tcp://minio:9000
      -wait tcp://redis:6379
      -timeout 120s
```

From pipeline start to first assertion: ninety seconds — assuming images are already cached on the runner; a cold pull adds time, but becomes negligible once the pipeline has run once on a given branch.

## What `$CI_COMMIT_REF_SLUG` means

The timing is the visible result. What produces it is more interesting: the image names.

`$REGISTRY_URL/platform/rabbitmq:$CI_COMMIT_REF_SLUG` is not the official RabbitMQ image from Docker Hub. It's an image built by the same pipeline, from the same branch, at the same commit as the code being tested. The RabbitMQ image carries the topology: a `definitions.json` with every exchange, every queue, every binding, every dead-letter configuration — versioned in git alongside the application that depends on them.

If a branch modifies the messaging topology, the CI pipeline builds a new RabbitMQ image that includes those modifications, then runs the tests against it. The topology change and the code that relies on it are tested together, at the same commit, before anything reaches staging.

The same logic applies to MinIO, as described in the [first article in this series](/2026/05/14/the-ghost-of-the-ci-runner/): the MinIO image carries preloaded test fixtures. The CI environment doesn't need a setup step to populate storage. The state is built in.

The test runner itself follows the same pattern. Each job uses a debug variant of the application image — built from the same branch, same commit — with the test dependencies included:

```yaml
image: $REGISTRY_URL/platform/$service:$CI_COMMIT_REF_SLUG-debug
```

The whole environment assembles from artifacts built at the same point in the git history.

## What this required dropping

Behat and the provisioned VM were coupled. The Behat test suite ran against an HTTP server on the VM; removing the VM meant removing Behat.

That turned out not to be the obstacle it looked like. The Behat suite lived in a separate repository, required the VM to run, and had accumulated significant maintenance overhead. PHPUnit, running inside the application container with Docker services, covered the same scenarios through a more direct path: functional tests exercising the HTTP layer, unit tests for individual components, suites organized per feature area and generated dynamically into parallel CI jobs.

The BDD layer went away. The test coverage stayed — and could now run against the actual services.

## Factor X, applied

[Factor X](https://12factor.net/dev-prod-parity) is often read as "use the same database locally as in production." That's the simplest version. The deeper version is about the gap between what you test and what you ship.

The gap in the old pipeline was wide: a manually configured VM, missing key services, rebuilt from scratch on every run. The gap in the new pipeline is narrow: the CI assembles the environment from the same images as production, built from the same commit as the code under test.

The fifteen minutes of Terraform and Ansible were not just slow. They were building something that wasn't what production ran, every time, before any test could begin. The ninety seconds of `docker pull` build exactly what production runs — and the tests that follow are testing that, not an approximation of it.
