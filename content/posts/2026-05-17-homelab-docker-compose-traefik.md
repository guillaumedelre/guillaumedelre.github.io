---
title: "Concevoir un homelab qui survit à ses services"
aliases:
  - "/fr/2026/05/17/construire-un-homelab-self-hosted-avec-docker-compose-et-traefik/"
  - "/2026/05/17/building-a-self-hosted-homelab-with-docker-compose-and-traefik/"
  - "/fr/2026/02/17/construire-un-homelab-self-hosted-avec-docker-compose-et-traefik/"
  - "/2026/02/17/building-a-self-hosted-homelab-with-docker-compose-and-traefik/"
date: 2026-05-17
lastmod: 2026-11-18
tags: [homelab, docker, traefik]
format: guide
description: "Guide de conception d'un homelab Docker : les principes qui permettent d'ajouter ou de retirer n'importe quel service sans toucher au reste."
---

{{< lead >}}
Dans la première version de cet article, je présentais dix services. La plupart n'étaient là que pour l'exemple. Aujourd'hui, dix-huit stacks tournent sur ma machine : une IA locale, Home Assistant, un serveur de musique, des applis que j'ai écrites moi-même. Aucune n'a demandé de toucher à l'architecture.
{{< /lead >}}

C'est ce qui m'intéresse dans un homelab : pas la liste des services, qui change tous les trois mois, mais la structure qui les accueille. Une bonne structure rend l'ajout d'un service banal et sa suppression sans conséquence. Cet article décrit les huit principes qui tiennent la mienne, avec à chaque fois la manière dont je les applique.

Il s'adresse à quelqu'un qui connaît Docker et veut faire tourner plusieurs services sur une machine du réseau local. Il faut une machine Linux et Docker avec Compose v2. Rien d'autre : ni nom de domaine, ni serveur DNS, ni VPS.

## 1. Un service, un dossier

Chaque service vit dans son propre dossier, avec son propre `compose.yaml` et son propre `.env`. Toutes les commandes se lancent depuis ce dossier :

```text
~/homelab/
├── traefik/      compose.yaml  .env  docker/
├── homepage/     compose.yaml  .env  docker/
├── dozzle/       compose.yaml  .env
├── hass/         compose.yaml  .env  docker/
└── …
```

Ça paraît évident, mais c'est le principe dont découlent tous les autres. Un dossier par service, c'est un service qu'on démarre, qu'on arrête, qu'on met à jour ou qu'on supprime sans risquer d'emporter le voisin. Le jour où j'ai ajouté une stack IA de quatre conteneurs (Ollama, Open WebUI, ChromaDB, Docling), je n'ai rien modifié ailleurs. Et retirer un service se résume à un `docker compose down` suivi de la suppression d'un dossier.

Un seul gros `compose.yaml` pour tout le homelab marche aussi, au début. Puis un `docker compose up -d` pour un service en redémarre trois autres, une erreur de syntaxe bloque tout, et l'historique git mélange tout.

Les secrets restent dans le `.env` de chaque dossier, jamais versionné. Quand une valeur n'est pas évidente, un `.env.example` à côté documente ce qu'il faut renseigner.

## 2. Une seule porte d'entrée

Aucun service ne publie de port sur la machine. Un seul conteneur le fait : le reverse proxy, [Traefik][traefik], qui écoute sur le port 80 et route chaque requête vers le bon conteneur selon le nom demandé.

{{< mermaid >}}
flowchart LR
  B[Navigateur] -->|dozzle.10.0.0.10.sslip.io| T[Traefik :80]
  T -->|réseau traefik| D[Dozzle]
  T -->|réseau traefik| H[Homepage]
  T -->|réseau traefik| A[Home Assistant]
  T -.->|host.docker.internal| M[Music Assistant, réseau de l'hôte]
{{< /mermaid >}}

Pour que Traefik joigne les conteneurs, tout le monde partage un réseau Docker, créé une fois pour toutes en dehors de tout Compose :

```bash
docker network create traefik
```

Chaque `compose.yaml`, y compris celui de Traefik, le déclare comme externe (`external: true`). C'est ce qui permet à des stacks indépendantes de se retrouver sur le même réseau : un réseau créé par un Compose n'est visible que de ses propres services.

Dans la configuration de Traefik, une ligne compte plus que les autres : `exposedByDefault: false`. Traefik ignore alors tout conteneur qui ne demande pas explicitement à être exposé. Sans elle, une base de données lancée pour un test se retrouve avec une URL.

### Les services qui refusent le réseau commun

Certains services doivent tourner sur le réseau de l'hôte (`network_mode: host`). Music Assistant en a besoin pour découvrir les enceintes en mDNS et UPnP, Glances pour voir les vraies interfaces et les vrais processus de la machine. Ils ne peuvent donc pas rejoindre le réseau `traefik`.

Traefik sait les joindre via `host.docker.internal`, à un détail près : Docker sous Linux ne définit pas ce nom. Traefik se rabat alors sur `127.0.0.1`, c'est-à-dire sur lui-même, et répond par une erreur 502. La correction tient en une ligne dans le `compose.yaml` de Traefik :

```yaml
extra_hosts:
    - "host.docker.internal:host-gateway"
```

Ces services gardent leurs labels habituels, mais sans la section `networks:` ni le label `traefik.docker.network`. C'est la seule exception de toute l'installation, et elle est écrite noir sur blanc dans les notes du homelab, avec la liste des services concernés.

## 3. Des noms sans serveur DNS

Pour router selon le nom, il faut des noms. Modifier `/etc/hosts` sur chaque machine ne tient pas dès qu'il y a un téléphone dans la maison, et monter un DNS local, c'est un service de plus à maintenir.

J'utilise [sslip.io][sslip], un DNS public qui renvoie l'adresse IP écrite dans le nom : `dozzle.10.0.0.10.sslip.io` pointe vers `10.0.0.10`. Ça fonctionne depuis n'importe quel appareil du réseau, sans rien configurer. L'adresse de la machine et le domaine sont dans le `.env` de chaque stack :

```dotenv
IP=10.0.0.10
DOMAIN=sslip.io
```

Reste à construire le nom de chaque service. Docker Compose fournit `COMPOSE_PROJECT_NAME`, qui vaut par défaut le nom du dossier. Si le dossier porte le nom du sous-domaine, il n'y a rien à déclarer : le dossier `dozzle/` répond sur `dozzle.10.0.0.10.sslip.io`. Quand ce n'est pas possible, j'écris le nom en dur dans la règle Traefik, et c'est la seule exception.

Pour vérifier ce que vaut la variable dans un dossier avant de s'en servir :

```bash
printf 'services:\n  t:\n    image: alpine\n    labels:\n      test: "${COMPOSE_PROJECT_NAME}"\n' \
  | docker compose --project-directory <dossier> -f - config
```

Le label `test` affiché doit être le sous-domaine attendu.

## 4. La configuration voyage avec le service

C'est le principe qui rend l'ajout d'un service banal. Tout ce que les autres briques doivent savoir d'un service est écrit dans son propre `compose.yaml`, sous forme de labels Docker : sa route pour Traefik, sa carte pour le tableau de bord. Traefik et [Homepage][homepage] lisent ces labels en direct. Ajouter un service ne modifie ni l'un ni l'autre.

Voici le `compose.yaml` complet de Dozzle, tel qu'il tourne chez moi. Tous les autres suivent le même modèle :

```yaml
services:
    dozzle:
        image: amir20/dozzle:latest
        restart: unless-stopped
        healthcheck:
            test: ["CMD", "/dozzle", "healthcheck"]
            interval: 30s
            timeout: 10s
            retries: 3
            start_period: 10s
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        networks:
            - traefik
        labels:
            homepage.group: Supervision
            homepage.name: Dozzle
            homepage.icon: https://cdn.jsdelivr.net/gh/selfhst/icons/webp/dozzle.webp
            homepage.href: http://${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}

            traefik.enable: true
            traefik.http.routers.dozzle.entrypoints: web
            traefik.http.routers.dozzle.rule: Host(`${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}`)
            traefik.http.services.dozzle.loadbalancer.server.port: 8080

networks:
    traefik:
        external: true
```

Les groupes de Homepage disent beaucoup de ce que devient un homelab avec le temps. Les miens s'appellent Famille, Maison, Loisirs, IA, Médias, Supervision et Infrastructure : l'infrastructure n'en est plus qu'un parmi d'autres. L'ordre des groupes, lui, se règle une fois dans la configuration de Homepage, et c'est le seul endroit à toucher quand on crée un nouveau groupe.

## 5. Voir ce qui tourne

Avec dix-huit stacks, on ne se souvient plus de ce qui tourne. Trois briques répondent à trois questions différentes :

- **Qu'est-ce qui existe ?** Homepage, qui se remplit tout seul grâce aux labels.
- **Que se passe-t-il ?** [Dozzle][dozzle], pour lire les logs de n'importe quel conteneur dans le navigateur.
- **Est-ce que la machine suit ?** [Glances][glances], pour le processeur, la mémoire, les disques et le réseau.

La quatrième brique ne se voit pas, et c'est la plus importante : un health check sur chaque service. Traefik ignore les conteneurs `unhealthy`, sans prévenir. Un service avec un health check mal écrit tourne parfaitement, mais n'a pas de route. Deux cas rencontrés en pratique :

- **`localhost` n'est pas toujours `127.0.0.1`.** Dans certaines images, `localhost` résout d'abord vers l'adresse IPv6 `::1`, alors que l'application n'écoute qu'en IPv4. Le health check échoue sans raison apparente. Écrire `127.0.0.1` règle le problème.
- **Certaines images n'ont pas de shell.** Les images construites sur `scratch`, comme Dozzle, n'ont ni `/bin/sh` ni `wget`. On utilise alors la sous-commande de santé du binaire lui-même : `["CMD", "/dozzle", "healthcheck"]`.

## 6. L'outil de pilotage est un détail

Pour démarrer, arrêter ou mettre à jour une stack, `docker compose` dans son dossier suffit. Par-dessus, j'utilise un petit outil maison, `stacks`, qui liste les projets Compose de la machine et ajoute des boutons aux cartes de Homepage : logs en direct, redémarrage, reconstruction.

L'important n'est pas l'outil, c'est qu'il soit remplaçable. Comme chaque service est un projet Compose ordinaire, n'importe quel outil sait le piloter : la ligne de commande, [Dockge][dockge], Portainer ou le mien. Le jour où j'arrêterai `stacks`, rien ne cassera. Un outil qui stockerait la configuration des services dans sa propre base, au lieu des fichiers, poserait le problème inverse : il deviendrait impossible de s'en passer.

## 7. Choisir sa politique de mise à jour

Toutes mes images sont en `:latest`, et [Watchtower][watchtower] les met à jour chaque nuit à 3 h, conteneur par conteneur, en supprimant les anciennes images. C'est l'inverse de ce qu'on recommande en production, et c'est un choix. Un homelab à jour sans effort vaut, pour moi, le risque d'une mise à jour qui casse un service un matin. Le principe 1 limite les dégâts à un seul dossier.

Ce choix a une date de péremption que je surveille :

{{< alert >}}
Le dépôt de Watchtower [a été archivé](https://github.com/containrrr/watchtower) en décembre 2025. L'image fonctionne toujours, mais elle ne reçoit plus de correctifs. Si Docker change encore son API minimale, il faudra le remplacer.
{{< /alert >}}

L'autre politique possible, c'est d'épingler chaque version et de mettre à jour à la main. Elle est plus sûre, et beaucoup plus lente. L'essentiel est d'en choisir une consciemment : la pire politique, c'est des versions épinglées qu'on ne met jamais à jour.

## 8. Savoir ce qu'on n'a pas fait

Mon homelab est conçu pour le réseau local, et seulement pour lui. Les limites sont assumées, mais je préfère les écrire :

- **Pas de HTTPS.** Tout passe en HTTP sur le port 80. Pour avoir des certificats sans domaine, j'ai décrit une solution avec mkcert dans [HTTPS local avec Traefik]({{< relref "2025-04-17-traefik-local-https-mkcert-sslip.md" >}}).
- **Pas d'authentification.** Plusieurs interfaces, comme le tableau de bord de Traefik ou [claude-agents-board]({{< relref "2026-11-17-claude-agents-board.md" >}}), sont ouvertes à tout appareil du réseau.
- **Le socket Docker est monté dans plusieurs conteneurs.** Traefik, Dozzle, Watchtower et `stacks` y ont accès. Un accès au socket équivaut à un accès root à la machine.

{{< alert >}}
Ces choix ne tiennent que si rien n'est exposé sur Internet. Pas de redirection de port sur la box, pas de tunnel vers ces services.
{{< /alert >}}

## Le test

Pour savoir si une architecture de homelab tient, j'ai un test simple : combien de fichiers faut-il toucher pour ajouter un service ? Chez moi, la réponse est deux, dans un nouveau dossier : un `compose.yaml` copié d'un voisin et un `.env` de deux lignes. Pour en retirer un, c'est zéro : on supprime le dossier.

Si votre réponse est plus longue, le problème n'est pas le service que vous essayez d'ajouter. Pour trouver de quoi remplir le vôtre, [selfh.st][selfhst] recense les logiciels auto-hébergés par catégorie, et [selfhst/icons][selfhst-icons] fournit les icônes utilisées dans les labels Homepage.

[sslip]: https://sslip.io
[selfhst]: https://selfh.st
[selfhst-icons]: https://github.com/selfhst/icons
[traefik]: https://github.com/traefik/traefik
[homepage]: https://github.com/gethomepage/homepage
[dozzle]: https://github.com/amir20/dozzle
[glances]: https://github.com/nicolargo/glances
[dockge]: https://github.com/louislam/dockge
[watchtower]: https://github.com/containrrr/watchtower
