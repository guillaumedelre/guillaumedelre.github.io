---
title: "From Vagrant to Docker Compose: a retrospective"
date: 2022-04-18
categories: [devops]
tags: [docker, vagrant, docker-compose, devops]
description: "Why we replaced Vagrant with Docker Compose: the real friction points, the migration path, and what we'd do differently."
---

I ran Vagrant for years. A Vagrantfile per project, a shared base box, a provision script that worked on Tuesday but not on Thursday. The promise was simple: reproducible environments for everyone on the team. The reality was more complicated.

## The Vagrant years

The setup made sense at the time. One VM per project, provisioned with shell scripts or Ansible, shared via a versioned Vagrantfile. Onboarding was theoretically `vagrant up` and you're done.

In practice, it was `vagrant up`, wait four minutes, watch the provision fail on a package that changed its download URL, fix it, reprovision, wait again. Vagrantfiles accumulated configuration over time: workarounds for specific machines, OS version pinning, memory tweaks for the team member whose laptop had 8GB. The files became historical documents nobody wanted to touch.

The VM itself was the other problem. Booting took time. Running took memory and CPU that could have gone to the application. File syncing between host and guest added latency that made PHP apps feel slower than they had any right to be. The overhead was significant for what was ultimately just "run a web server."

We lived with it because everyone did. Vagrant was the standard for local PHP development, and the alternative (each developer managing their own LAMP stack) was clearly worse.

## The project that changed the model

The shift wasn't a decision we made. It was a project that arrived already containerized.

A new client project had a `docker-compose.yml` at the root, a `Dockerfile`, and a README that said `docker compose up`. We ran it. The containers started in seconds. PHP-FPM, nginx, PostgreSQL, Redis: all running, all networked, no provisioning step. Stop the containers, start them again, same state.

The contrast with our Vagrant setup was immediate. Not faster by a percentage: faster by a different order. And the Compose file was actually readable: each service, its image, its volumes, its environment variables, its dependencies. Compared to a provision script that SSHed into a VM and ran apt-get, this was legible.

We migrated everything. Not gradually, all at once, over a sprint. Every project got a `docker-compose.yml`. Every Vagrantfile was deleted. The transition was the most painful three weeks of infrastructure work I remember, and also the most clearly worth it.

## What docker-compose actually changed

Beyond the speed, Compose changed the mental model. Vagrant abstracted a machine. Compose abstracted a set of processes. The distinction matters: with Compose, you can stop the database without stopping the application server, scale a worker service independently, swap the PostgreSQL image for a newer version without touching anything else.

The `services` declaration also replaced the VM provisioning problem entirely. If a new developer joins, they don't run a provision script that may or may not work on their OS version. They run `docker compose up` and get the exact same images everyone else runs.

CI/CD got simpler too. The same `docker-compose.yml` that ran locally could run in the pipeline. The environment parity that Vagrant promised but rarely delivered was actually real with Compose.

## The quiet deprecation

For years, the command was `docker-compose`: a separate binary, installed independently from Docker itself, written in Python, versioned independently. We used it, it worked, nobody thought much about it.

At some point a colleague mentioned that Docker had integrated Compose directly into the `docker` CLI. The new command was `docker compose`, no hyphen, Go rewrite, bundled with Docker Desktop. The old `docker-compose` binary was deprecated.

We had been using v1 for two years after v2 shipped. Our CI scripts, our Makefiles, our documentation all said `docker-compose`. Nothing had broken because Docker maintained the old binary for a long time. But the ecosystem had moved on quietly, and we'd missed it.

The migration was trivial: a hyphen removed from every script, a few aliases updated. The lesson was less trivial. Infrastructure tooling evolves without ceremony. The announcement happened, the blog posts were written, the deprecation notices were there. We just weren't paying attention.

## The actual retrospective

Looking back across Vagrant → `docker-compose` → `docker compose`, the pattern is less about the tools and more about the defaults.

Vagrant defaulted to "it works on my VM." The overhead of sharing that VM was permanent.

Compose defaulted to "it works in these containers." The images are the artifacts; the host machine is irrelevant.

The hyphen between `docker` and `compose` was always cosmetic. What mattered was the shift from provisioned machines to declarative services. That shift happened the day we ran a project someone else containerized and realized we never wanted to go back.
