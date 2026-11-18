---
title: "Construire un homelab self-hosted avec Docker Compose et Traefik"
aliases: ["/fr/2026/02/17/construire-un-homelab-self-hosted-avec-docker-compose-et-traefik/"]
date: 2026-05-17
lastmod: 2026-11-17
tags: [homelab, docker, traefik]
format: guide
description: "Tuto complet pour monter un homelab Docker avec Traefik et sslip.io : stacks indépendantes, dashboard auto-configuré, pièges documentés."
---

Ça fait des années que j'avais envie d'un homelab à la maison. Un endroit à moi pour héberger mes outils de développement, surveiller mes machines, faire tourner de la domotique, tester des trucs sans risquer de casser quoi que ce soit d'important. L'idée est simple. La mise en place un peu moins.

À l'époque, Kubernetes n'existait pas encore. Les options pour faire tourner plusieurs services sur une machine se résumaient à du scripting bash, des configurations Nginx écrites à la main, et beaucoup de café. Les tutoriels « homelab pour les humains » brillaient par leur absence.

Ce tuto, c'est ce que j'aurais voulu trouver à l'époque. Ça tourne depuis plusieurs années maintenant. Pas sans évoluer : des services ajoutés, d'autres abandonnés, des choix revisités. Mais la base est là, stable, et c'est bien ça le succès en self-hosting.

Le setup : dix services auto-hébergés sur une machine locale, accessibles depuis un navigateur via des URLs lisibles, sans toucher à la configuration DNS, sans louer un VPS, sans certificat TLS à gérer. L'ingrédient qui rend ça possible : [sslip.io][sslip], un service DNS public qui encode l'IP directement dans le nom de domaine. `service.192.168.1.10.sslip.io` résout vers `192.168.1.10`, sans rien configurer, depuis n'importe quelle machine du réseau local.

Ce tutoriel s'adresse à quelqu'un qui connaît Docker mais qui part de zéro sur l'orchestration de services self-hosted.

---

## Table des matières

1. [Philosophie et choix d'architecture](#1-philosophie-et-choix-darchitecture)
2. [Les briques fondamentales](#2-les-briques-fondamentales)
3. [Mise en place pas à pas](#3-mise-en-place-pas-à-pas)
4. [Ajouter un nouveau service](#4-ajouter-un-nouveau-service)
5. [Patterns et conventions](#5-patterns-et-conventions)
6. [Pièges courants](#6-pièges-courants)
7. [Conclusion](#conclusion)
8. [Références](#références)

---

## 1. Philosophie et choix d'architecture

### Objectif

Faire tourner plusieurs services web sur une machine locale, accessibles depuis un navigateur via des URLs lisibles, sans toucher à la configuration DNS, sans louer un VPS, sans certificat TLS à gérer.

### Pourquoi Docker Compose et pas autre chose ?

Docker Compose est le bon niveau de complexité pour un homelab personnel. Kubernetes est trop lourd pour une seule machine. Docker Swarm est en déclin. Compose est simple, lisible, versionnable, et suffisant pour des dizaines de services.

### Pourquoi Traefik et pas Nginx Proxy Manager ?

**Nginx Proxy Manager (NPM)** est une interface graphique pour configurer Nginx comme reverse proxy. Les routes sont stockées dans une base de données et configurées via une UI.

**[Traefik][traefik]** lit automatiquement les labels Docker des containers et génère sa configuration à la volée. Quand on démarre un container avec les bons labels, Traefik le découvre et crée la route immédiatement, sans redémarrage, sans UI à ouvrir.

Ce comportement « configuration as code » a deux avantages majeurs :
- La configuration d'un service est dans son `compose.yaml`, au même endroit que tout le reste.
- Ajouter un service ne nécessite pas de toucher à Traefik.

### Pourquoi Dockge et pas Portainer ?

**Portainer** est un outil de gestion Docker complet : images, volumes, réseaux, containers individuels... puissant mais complexe.

**[Dockge][dockge]** est focalisé sur une seule chose : gérer des stacks Docker Compose. Son UI est minimaliste et intuitive. Pour un homelab où tout est géré en Compose, c'est suffisant et bien plus agréable à utiliser.

### Pourquoi sslip.io ?

Les services web ont besoin d'un nom d'hôte (ex. : `dozzle.monserveur.local`) pour que Traefik puisse les router correctement. Les options habituelles :
- Modifier `/etc/hosts` sur chaque machine : fastidieux, non partageable.
- Configurer un vrai DNS local (Pi-hole, AdGuard) : nécessite une infrastructure supplémentaire.
- Acheter un domaine et configurer les DNS : coûte de l'argent et du temps.

**sslip.io** est un service DNS public qui résout automatiquement `<anything>.<IP>.sslip.io` vers `<IP>`. Exemple : `dozzle.192.168.1.10.sslip.io` résout vers `192.168.1.10`. Il n'y a rien à configurer, le DNS fonctionne partout sans toucher à quoi que ce soit.

---

## 2. Les briques fondamentales

### Le réseau Docker partagé

Tous les services et Traefik doivent partager le même réseau Docker pour que Traefik puisse communiquer avec eux. Ce réseau s'appelle `traefik` et est créé une seule fois :

```bash
docker network create traefik
```

C'est un réseau **externe** (créé hors de tout Compose). Chaque `compose.yaml` le déclare comme externe :

```yaml
networks:
    traefik:
        external: true
```

Pourquoi externe plutôt qu'interne à un Compose ? Parce que plusieurs stacks indépendantes doivent toutes y être connectées. Un réseau interne à un Compose n'est accessible qu'aux services de ce Compose.

### Traefik : le reverse proxy

Traefik écoute sur le port 80 et route les requêtes HTTP vers le bon container selon le `Host` header. Son `compose.yaml` publie aussi les ports 443 et 25565, mais aucun entrypoint ne les utilise encore : ils sont réservés pour plus tard.

Sa configuration principale est dans `stacks/traefik/docker/traefik/traefik.yaml` :

```yaml
api:
    dashboard: true
    insecure: true
    debug: true

entryPoints:
    web:
        address: :80
        forwardedHeaders:
            insecure: true
    ping:
        address: ":8082"

ping:
    entryPoint: ping

providers:
    docker:
        endpoint: unix:///var/run/docker.sock
        exposedByDefault: false

global:
    sendAnonymousUsage: false

log:
    level: WARN
```

`exposedByDefault: false` est important : Traefik ignore tous les containers par défaut. Un container doit explicitement s'exposer avec le label `traefik.enable: true`. Cela évite d'exposer accidentellement des services.

L'entrypoint `ping` sur le port 8082 est dédié aux health checks, et c'est le bloc `ping` de premier niveau qui sert réellement `/ping` dessus. Sans ce bloc, l'endpoint n'existe pas, le health check échoue et le container de Traefik lui-même reste `unhealthy`. Le garder à part de l'entrypoint `web` évite aussi que les checks apparaissent dans les logs d'accès.

Pour accéder au daemon Docker, Traefik monte le socket :

```yaml
volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

### Dockge : le gestionnaire de stacks

Dockge tourne lui-même dans un container (le `compose.yaml` à la racine du repo). Il a besoin de deux choses :
1. Accès au socket Docker pour piloter les autres containers.
2. Accès aux dossiers des stacks pour lire et modifier les `compose.yaml`.

Le point critique est le montage des stacks. Dockge lance les stacks en passant des chemins absolus au daemon Docker. Ces chemins doivent être identiques dans le container Dockge et sur le host. La solution :

```yaml
volumes:
    - ${PWD}/stacks:${PWD}/stacks
environment:
    DOCKGE_STACKS_DIR: ${PWD}/stacks
```

`${PWD}` est une variable shell résolue au moment du `docker compose up`. Elle vaut le répertoire courant. Si on lance Dockge depuis `/home/user/homelab`, le dossier stacks sera monté à `/home/user/homelab/stacks` des deux côtés. C'est la seule façon d'éviter que Docker crée des répertoires fantômes au mauvais endroit.

**Conséquence pratique** : toujours lancer `docker compose up -d` depuis la racine du repo.

La donnée persistante de Dockge (configuration, historique) est dans un volume nommé créé à l'avance :

```bash
docker volume create homelab_dockge_data
```

Comme il est déclaré `external: true`, un `docker compose down -v` n'y touche pas. Un volume créé par Compose lui-même, nommé ou anonyme, serait supprimé avec la stack.

---

## 3. Mise en place pas à pas

### Étape 1 : cloner et configurer

```bash
git clone <repo> homelab
cd homelab
```

Trouver l'IP locale de la machine :

```bash
hostname -I | awk '{print $1}'
# ex: 192.168.1.10
```

Créer et éditer le `.env` racine :

```bash
cp .env.example .env
# Éditer .env :
# IP=192.168.1.10
# DOMAIN=sslip.io
# COMPOSE_PROJECT_NAME=dockge  ← important, voir section conventions
```

### Étape 2 : prérequis Docker

```bash
docker network create traefik
docker volume create homelab_dockge_data
```

### Étape 3 : démarrer Dockge

```bash
docker compose up -d
```

Dockge est accessible sur `http://<IP>:5001`. Il est exposé directement sur le port 5001, pas via Traefik (Traefik n'est pas encore démarré à ce stade). Créer un compte admin à la première ouverture.

### Étape 4 : configurer les stacks

Pour chaque dossier dans `stacks/`, copier le `.env.example` (Watchtower n'en a pas, il n'a besoin d'aucune variable) :

```bash
for stack in stacks/*/; do
    [ -f "${stack}.env.example" ] && cp "${stack}.env.example" "${stack}.env"
done
```

Puis éditer chaque `.env` pour renseigner `IP` et `DOMAIN` avec les mêmes valeurs qu'à l'étape 1. La plupart ne contiennent rien d'autre. `filebrowser` et `uptime-kuma` définissent aussi `COMPOSE_PROJECT_NAME` avec le nom de leur dossier : ne pas y toucher (voir section conventions).

Pour `filebrowser`, renseigner aussi `FILEBROWSER_ROOT` avec le chemin local à exposer.

### Étape 5 : lancer les stacks depuis Dockge

Depuis l'interface Dockge (`http://<IP>:5001`), dans cet ordre :

**1. Traefik en premier**

Traefik doit être actif avant les autres services. Sans Traefik, les routes n'existent pas et les services sont inaccessibles via leur URL.

Après démarrage, vérifier que Traefik est healthy :

```bash
docker ps --filter name=traefik
```

**2. Les autres stacks dans n'importe quel ordre**

Chaque stack se déclare automatiquement auprès de Traefik via ses labels Docker. Traefik découvre les nouveaux containers en temps réel.

Homepage n'a pas besoin d'une place particulière : il interroge les labels Docker des containers actifs à chaque chargement du dashboard, donc un service démarré après lui apparaît au rafraîchissement suivant.

---

## 4. Ajouter un nouveau service

Voici le template de `compose.yaml` pour tout nouveau service :

```yaml
services:
    monservice:
        image: editeur/monservice:latest
        restart: unless-stopped
        healthcheck:
            test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:<PORT>/ || exit 1"]
            interval: 30s
            timeout: 10s
            retries: 3
            start_period: 10s
        labels:
            # Homepage - apparition automatique dans le dashboard
            homepage.group: outils
            homepage.name: Mon Service
            homepage.icon: https://cdn.jsdelivr.net/gh/selfhst/icons/webp/monservice.webp
            homepage.href: http://${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}

            # Traefik - routage HTTP
            traefik.enable: true
            traefik.http.routers.monservice.entrypoints: web
            traefik.http.routers.monservice.rule: Host(`${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}`)
            traefik.http.services.monservice.loadbalancer.server.port: <PORT>
        networks:
            - traefik

networks:
    traefik:
        external: true
```

Et le `.env.example` associé :

```
IP=127.0.0.1
DOMAIN=sslip.io
```

**Le nom du dossier détermine le sous-domaine.** Si le dossier s'appelle `monservice`, le service sera accessible sur `monservice.<IP>.<DOMAIN>`. C'est tout.

Pour trouver des services à ajouter, [selfh.st][selfhst] est une excellente ressource : c'est un catalogue de logiciels self-hosted organisé par catégorie (media, sécurité, productivité, monitoring...), avec pour chacun une description, une capture d'écran et le lien GitHub. Le site publie aussi une newsletter hebdomadaire sur les nouvelles releases.

### Checklist pour un nouveau service

- [ ] Créer `stacks/<nom-du-sous-domaine>/compose.yaml`
- [ ] Créer `stacks/<nom-du-sous-domaine>/.env.example` avec `IP` et `DOMAIN`
- [ ] Copier `.env.example` en `.env` et renseigner IP/DOMAIN
- [ ] Vérifier le port dans les labels Traefik
- [ ] Choisir le groupe Homepage : `infra`, `observabilité`, ou `outils`
- [ ] Trouver l'icône sur [selfhst/icons][selfhst-icons]
- [ ] Ajouter les données persistantes dans un volume si nécessaire
- [ ] Lancer depuis Dockge et vérifier que le container est `healthy`

---

## 5. Patterns et conventions

### La variable `${COMPOSE_PROJECT_NAME}`

Docker Compose valorise automatiquement `COMPOSE_PROJECT_NAME` avec le nom du dossier qui contient le `compose.yaml`, ici le dossier de la stack. On l'utilise pour construire dynamiquement les URLs :

```yaml
traefik.http.routers.dozzle.rule: Host(`${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}`)
homepage.href: http://${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}
```

Avantage : pas de variable `*_HOST` à maintenir dans chaque `.env`. Renommer le dossier change automatiquement le sous-domaine.

**L'exception, c'est la racine.** Le dossier du repo s'appelle `homelab`, mais Dockge doit répondre sur `dockge.<IP>.<DOMAIN>`. C'est pour ça que le `.env` racine définit explicitement `COMPOSE_PROJECT_NAME=dockge`. Les stacks n'en ont pas besoin ; les deux qui le définissent (`filebrowser`, `uptime-kuma`) ne font que répéter le nom de leur dossier.

### Les groupes Homepage

Les services sont organisés en trois groupes dans le dashboard :

| Groupe | Services |
|---|---|
| `infra` | [Traefik][traefik], [Dockge][dockge], [Watchtower][watchtower], [Homepage][homepage] |
| `observabilité` | [Dozzle][dozzle], [Glances][glances], [Uptime Kuma][uptime-kuma] |
| `outils` | [FileBrowser][filebrowser], [IT-Tools][ittools], [Stirling PDF][stirling-pdf] |

Ce découpage est celui de ce homelab, pas une convention imposée. Homepage accepte n'importe quelle valeur dans `homepage.group` : on peut créer autant de groupes que nécessaire et les nommer comme on veut (`media`, `domotique`, `dev`...). Le dashboard se réorganise automatiquement.

### Health checks

Tous les services ont un health check. C'est crucial car **Traefik ignore silencieusement les containers `unhealthy`** : un service avec un health check défaillant n'apparaît pas dans le routage, même avec `traefik.enable: true`.

Trois cas particuliers rencontrés en pratique :

**1. `localhost` ne veut pas toujours dire `127.0.0.1`**

Dans certaines images, `localhost` résout d'abord vers l'adresse IPv6 `::1`, alors que l'application n'écoute qu'en IPv4. Le health check échoue alors sans raison apparente. Utiliser `127.0.0.1` explicitement :

```yaml
test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/ || exit 1"]
```

**2. Images sans shell (`scratch`-based)**

Les images basées sur `scratch` (ex. : Dozzle) ne contiennent pas `/bin/sh`. `CMD-SHELL` échoue. Utiliser le binaire embarqué :

```yaml
test: ["CMD", "/dozzle", "healthcheck"]
```

**3. Images sans `wget` ni `curl`**

Certaines images Node.js ou JVM n'ont ni wget ni curl. Solutions possibles :
- Si Node.js est disponible : `node -e "require('http').get('http://localhost:PORT', r => process.exit(r.statusCode < 400 ? 0 : 1)).on('error', () => process.exit(1))"`
- Si curl est disponible : `curl -fs http://127.0.0.1:PORT/`
- Si le binaire de l'app expose une sous-commande healthcheck : l'utiliser directement.

### Persistance des données

Pour les services qui ont des données (configuration, base utilisateurs, base de données) :

```yaml
volumes:
    - ./docker/data:/chemin/dans/container
```

Le dossier `./docker/` est dans le dossier de la stack et peut être versionné, à l'exception des données runtime qui vont dans `.gitignore`.

**Règle** : ajouter `stacks/<service>/docker/` dans `.gitignore` si le dossier contient des données qui ne doivent pas être committées (base SQLite, uploads...).

### Organisation des labels Traefik

Par convention, le nom utilisé dans les labels Traefik (`traefik.http.routers.<nom>`) correspond au nom du service Docker dans le `compose.yaml`. En pratique on les aligne avec le nom du dossier :

```
stacks/it-tools/    →    service: ittools    →    traefik.http.routers.ittools.*
```

Ce n'est pas une contrainte technique de Traefik, juste une convention de lisibilité.

---

## 6. Pièges courants

### Dockge : Stop puis Start, pas Restart

Quand on modifie un `compose.yaml` depuis l'IDE et qu'on veut appliquer les changements, il faut faire **Stop + Start** depuis Dockge, pas « Restart ». Le Restart redémarre le container existant sans relire le `compose.yaml`. Le Stop + Start recrée le container avec la nouvelle configuration.

### Les labels modifiés n'apparaissent pas dans Homepage

Homepage relit les labels Docker à chaque chargement de page, donc un `homepage.group` ou `homepage.name` modifié doit apparaître après un rafraîchissement. Sinon, c'est que le container porte encore les anciens labels : il a été redémarré, pas recréé (voir Stop + Start plus haut).

### Le container démarre mais n'est pas routable

Vérifier dans l'ordre :

1. `docker ps` : le container est-il `healthy` ? Traefik ignore les containers `unhealthy`.
2. Le container est-il sur le réseau `traefik` ?

```bash
docker inspect <container> --format '{{json .NetworkSettings.Networks}}'
```

3. Le label `traefik.enable: true` est-il présent ?
4. La règle `Host(...)` correspond-elle à l'URL testée ?

### Monter un fichier qui n'existe pas encore

Quand un bind mount pointe vers un **fichier** qui n'existe pas encore sur le host, Docker crée un **répertoire** à la place. Ça arrive avec Docker Engine comme avec Docker Desktop sous WSL. Ce répertoire fantôme bloque ensuite le montage du vrai fichier. Symptôme : le container refuse de démarrer avec une erreur de montage.

Solution : s'assurer que le fichier existe sur le host avant de démarrer le container, ou utiliser un montage de répertoire plutôt que de fichier.

### Watchtower : API Docker trop ancienne

Sur certaines configurations, Watchtower tente de communiquer avec le daemon en commençant la négociation à l'API v1.25 (son minimum historique). Les versions récentes de Docker refusent cette version. Symptôme : le container redémarre en boucle avec `client version 1.25 is too old. Minimum supported API version is 1.40`.

Le `compose.yaml` de Watchtower du repo force la version :

```yaml
environment:
    DOCKER_API_VERSION: "1.40"
```

La valeur à mettre est le minimum indiqué dans le message d'erreur, pas votre version exacte. `1.40` suffisait quand le repo a été écrit, mais [Docker 29 a relevé le minimum à 1.44][watchtower-api] : à partir de Docker 29, mettre `1.44`. Pour vérifier la version d'API réelle de votre daemon :

```bash
docker version --format '{{.Server.APIVersion}}'
```

À noter : le dépôt de Watchtower [a été archivé][watchtower] en décembre 2025. L'image fonctionne toujours, mais elle ne reçoit plus de correctifs. À garder en tête avant de lui confier les mises à jour automatiques.

### `${PWD}` dans le compose de Dockge

`${PWD}` n'est pas une variable `.env`, c'est une variable shell résolue au moment du `docker compose up`. Elle vaut le répertoire courant du terminal. Lancer `docker compose up -d` depuis n'importe quel autre répertoire donnera une mauvaise valeur et cassera les montages de volumes des stacks.

---

*Ce homelab est conçu pour tourner sur une machine Linux ou WSL. Toutes les commandes sont testées sur Ubuntu/WSL2 avec Docker Desktop.*

---

## Conclusion

J'ai bien conscience que ce tuto ne couvre pas tout. On aurait pu ajouter de l'authentification devant chaque service, faire tourner l'ensemble en HTTPS, mettre en place un socket proxy pour limiter l'exposition du daemon Docker, ou épingler précisément chaque version d'image. Mais chacun de ces points aurait considérablement allongé l'article et la complexité de mise en place. L'objectif était de démarrer avec quelque chose de fonctionnel et maintenable, pas de construire une forteresse dès le premier jour.

Le homelab parfait n'existe pas. Celui qui tourne, si.

<a class="gh-card" href="https://github.com/guillaumedelre/homelab" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/homelab</span>
  </span>
  <span class="gh-card__desc">Homelab Docker Compose avec Traefik — stacks indépendantes, dashboard auto-configuré, et zéro configuration DNS grâce à sslip.io.</span>
</a>

## Références

| Projet | GitHub |
|---|---|
| sslip.io | [sslip.io][sslip] |
| selfh.st | [selfh.st][selfhst] |
| Traefik | [github.com/traefik/traefik][traefik] |
| Dockge | [github.com/louislam/dockge][dockge] |
| Homepage | [github.com/gethomepage/homepage][homepage] |
| Dozzle | [github.com/amir20/dozzle][dozzle] |
| Glances | [github.com/nicolargo/glances][glances] |
| FileBrowser | [github.com/gtsteffaniak/filebrowser][filebrowser] |
| IT-Tools | [github.com/CorentinTh/it-tools][ittools] |
| Stirling PDF | [github.com/Stirling-Tools/Stirling-PDF][stirling-pdf] |
| Uptime Kuma | [github.com/louislam/uptime-kuma][uptime-kuma] |
| Watchtower | [github.com/containrrr/watchtower][watchtower] |
| selfhst/icons | [github.com/selfhst/icons][selfhst-icons] |

[sslip]: https://sslip.io
[selfhst]: https://selfh.st
[traefik]: https://github.com/traefik/traefik
[dockge]: https://github.com/louislam/dockge
[homepage]: https://github.com/gethomepage/homepage
[dozzle]: https://github.com/amir20/dozzle
[glances]: https://github.com/nicolargo/glances
[filebrowser]: https://github.com/gtsteffaniak/filebrowser
[ittools]: https://github.com/CorentinTh/it-tools
[stirling-pdf]: https://github.com/Stirling-Tools/Stirling-PDF
[uptime-kuma]: https://github.com/louislam/uptime-kuma
[watchtower]: https://github.com/containrrr/watchtower
[selfhst-icons]: https://github.com/selfhst/icons
[watchtower-api]: https://github.com/containrrr/watchtower/issues/2124
