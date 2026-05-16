---
title: "De Vagrant à Docker Compose : une rétrospective"
date: 2022-04-18
categories: [devops]
tags: [docker, vagrant, docker-compose, devops, retrospective]
description: "Pourquoi on a remplacé Vagrant par Docker Compose : les vrais points de friction, le chemin de migration, et ce qu'on ferait différemment."
---

J'ai utilisé Vagrant pendant des années. Un Vagrantfile par projet, une box de base partagée, un script de provision qui marchait le mardi mais pas le jeudi. La promesse était simple : des environnements reproductibles pour tout le monde dans l'équipe. La réalité était plus compliquée.

## Les années Vagrant

Le setup avait du sens à l'époque. Une VM par projet, provisionnée avec des scripts shell ou Ansible, partagée via un Vagrantfile versionné. L'onboarding était théoriquement `vagrant up` et c'est terminé.

En pratique, c'était `vagrant up`, attendre quatre minutes, regarder la provision échouer sur un package qui avait changé son URL de téléchargement, corriger, reprovisionner, attendre à nouveau. Les Vagrantfiles accumulaient de la configuration au fil du temps : des contournements pour des machines spécifiques, du pinning de version d'OS, des ajustements mémoire pour le membre de l'équipe dont le laptop n'avait que 8 Go. Les fichiers devenaient des documents historiques que personne ne voulait toucher.

La VM elle-même était l'autre problème. Le boot prenait du temps. Faire tourner la VM consommait de la mémoire et du CPU qui auraient pu aller à l'application. La synchronisation des fichiers entre host et guest ajoutait une latence qui faisait paraître les applications PHP plus lentes qu'elles n'avaient le droit d'être. L'overhead était significatif pour ce qui était finalement juste "faire tourner un serveur web".

On vivait avec parce que tout le monde le faisait. Vagrant était le standard pour le développement PHP local, et l'alternative (chaque développeur gérant son propre stack LAMP) était clairement pire.

## Le projet qui a changé le modèle

Le changement n'était pas une décision qu'on a prise. C'était un projet qui est arrivé déjà conteneurisé.

Un nouveau projet client avait un `docker-compose.yml` à la racine, un `Dockerfile`, et un README qui disait `docker compose up`. On l'a lancé. Les conteneurs ont démarré en secondes. PHP-FPM, nginx, PostgreSQL, Redis : tout tournait, tout était en réseau, pas d'étape de provision. Arrêter les conteneurs, les redémarrer, même état.

Le contraste avec notre setup Vagrant était immédiat. Pas plus rapide d'un pourcentage : plus rapide d'un ordre de grandeur différent. Et le fichier Compose était réellement lisible : chaque service, son image, ses volumes, ses variables d'environnement, ses dépendances. Comparé à un script de provision qui SSH-ait dans une VM et lançait apt-get, c'était lisible.

On a tout migré. Pas progressivement, tout à la fois, sur un sprint. Chaque projet a reçu un `docker-compose.yml`. Chaque Vagrantfile a été supprimé. La transition a été les trois semaines de travail d'infrastructure les plus douloureuses dont je me souvienne, et aussi les plus clairement valables.

## Ce que docker-compose a vraiment changé

Au-delà de la vitesse, Compose a changé le modèle mental. Vagrant abstrayait une machine. Compose abstrayait un ensemble de processus. La distinction compte : avec Compose, on peut arrêter la base de données sans arrêter le serveur d'application, scaler un service worker indépendamment, échanger l'image PostgreSQL contre une version plus récente sans toucher à quoi que ce soit d'autre.

La déclaration `services` a aussi entièrement remplacé le problème de provision des VMs. Si un nouveau développeur rejoint l'équipe, il ne lance pas un script de provision qui peut ou ne pas fonctionner sur sa version d'OS. Il lance `docker compose up` et obtient exactement les mêmes images que tout le monde.

Le CI/CD est devenu plus simple aussi. Le même `docker-compose.yml` qui tournait en local pouvait tourner dans le pipeline. La parité d'environnement que Vagrant promettait mais livrait rarement était réellement réelle avec Compose.

## La dépréciation silencieuse

Pendant des années, la commande était `docker-compose` : un binaire séparé, installé indépendamment de Docker lui-même, écrit en Python, versionné indépendamment. On l'utilisait, ça marchait, personne n'y pensait vraiment.

À un moment, un collègue a mentionné que Docker avait intégré Compose directement dans le CLI `docker`. La nouvelle commande était `docker compose`, sans tiret, réécriture en Go, intégré avec Docker Desktop. L'ancien binaire `docker-compose` était déprécié.

On avait utilisé v1 pendant deux ans après que v2 était sortie. Nos scripts CI, nos Makefiles, notre documentation disaient tous `docker-compose`. Rien n'avait cassé parce que Docker avait maintenu l'ancien binaire longtemps. Mais l'écosystème avait évolué silencieusement, et on l'avait raté.

La migration était triviale : un tiret retiré de chaque script, quelques alias mis à jour. La leçon était moins triviale. Les outils d'infrastructure évoluent sans cérémonie. L'annonce avait eu lieu, les articles de blog avaient été écrits, les notices de dépréciation étaient là. On ne faisait juste pas attention.

## La vraie rétrospective

En regardant en arrière à travers Vagrant → `docker-compose` → `docker compose`, le pattern concerne moins les outils que les defaults.

Vagrant defaultait à "ça marche sur ma VM". L'overhead de partager cette VM était permanent.

Compose defaultait à "ça marche dans ces conteneurs". Les images sont les artefacts ; la machine host est hors sujet.

Le tiret entre `docker` et `compose` a toujours été cosmétique. Ce qui comptait, c'était le passage des machines provisionnées aux services déclaratifs. Ce passage a eu lieu le jour où on a lancé un projet que quelqu'un d'autre avait conteneurisé et où on a réalisé qu'on ne voulait jamais revenir en arrière.
