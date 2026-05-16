---
title: "HTTPS local avec Traefik: traefik.me est mort, vive sslip.io"
date: 2025-04-17
categories: [devops]
tags: [docker, traefik, mkcert, tls, https-local]
description: "Le certificat wildcard de traefik.me a été révoqué en 2025. Voici comment le remplacer avec sslip.io, mkcert et une configuration Traefik locale."
---

La configuration semblait parfaite. Pointer `*.traefik.me` sur 127.0.0.1, télécharger un certificat wildcard depuis le même domaine, le déposer dans Traefik, et chaque service local obtient une URL HTTPS propre sans IP dans la barre d'adresse. Pas de limites de débit Let's Encrypt, pas de `mkcert` à expliquer aux collègues, pas d'avertissements de certificat auto-signé à contourner. Juste `https://myapp.traefik.me` et un cadenas vert.

Puis en mars 2025, Let's Encrypt a révoqué le certificat. Le wildcard cert pour traefik.me est parti et il ne reviendra pas.

## Ce que traefik.me vendait vraiment

traefik.me est un résolveur DNS wildcard. Tapez `anything.traefik.me` et ça résout vers 127.0.0.1. Tapez `anything.10.0.0.1.traefik.me` et ça résout vers 10.0.0.1. Aucun compte, aucune configuration, aucune infrastructure à maintenir. La partie DNS fonctionne toujours bien, soit dit en passant.

Le certificat était le bonus: un wildcard cert pour `*.traefik.me` que pyrou, le mainteneur, avait généré avec Let's Encrypt et distribué sur `https://traefik.me/cert.pem` et `https://traefik.me/privkey.pem`. C'était pratique précisément parce que c'était partagé: télécharger, déposer dans Traefik, terminé.

Partager une clé privée, c'est ce qui l'a tué.

Les Baseline Requirements du CA/Browser Forum, section 9.6.3, exigent que les souscripteurs "maintiennent le contrôle exclusif" de leur clé privée. La distribuer à quiconque visite une URL, c'est exactement le contraire du contrôle exclusif. Let's Encrypt a envoyé une notification, bloqué toute future émission pour le domaine, et révoqué le certificat existant. Pyrou a confirmé la situation et recommandé mkcert comme alternative. Le projet survivra uniquement en tant que résolveur DNS.

Le cert avait déjà été révoqué deux fois avant 2025. La troisième était la dernière.

## sslip.io fait la même chose, différemment

sslip.io est aussi un résolveur DNS wildcard, avec une différence: l'IP est encodée dans le hostname plutôt que résolue depuis un fallback. `10-0-0-1.sslip.io` résout vers `10.0.0.1`. `myapp.192-168-1-10.sslip.io` résout vers `192.168.1.10`. IPv6 fonctionne aussi.

L'infrastructure derrière sslip.io est aussi plus visible: trois serveurs de noms à Singapour, aux États-Unis et en Pologne, traitant plus de 10 000 requêtes par seconde, avec un monitoring public. Environ 1 000 étoiles GitHub et une maintenance active sous licence Apache 2.0.

En mettant de côté l'histoire des certificats, la comparaison est assez directe:

| | traefik.me | sslip.io |
|---|---|---|
| DNS wildcard | oui | oui |
| Fallback vers 127.0.0.1 | oui | non |
| IPv6 | non | oui |
| Certificat wildcard | ~~oui~~ révoqué | non |
| Infrastructure | opaque | documentée |
| Activité du projet | au point mort | active |

Le seul avantage restant de traefik.me est le fallback vers 127.0.0.1: des URLs sans segment IP. Ça compte si on tient vraiment à `myapp.traefik.me` plutôt que `myapp.127-0-0-1.sslip.io`. La question est de savoir si cette différence vaut l'incertitude sur l'infrastructure.

## mkcert comble le vide

mkcert crée une autorité de certification locale, l'installe dans le trust store système et dans les navigateurs qu'il trouve, puis émet des certificats signés par cette CA. Les navigateurs voient une chaîne de confiance valide. Aucun avertissement, aucun clic, aucun "continuer quand même".

```bash
mkcert -install
```

C'est la configuration unique. Ensuite, générer un certificat se fait en une commande:

```bash
mkcert "*.127-0-0-1.sslip.io"
# produit _wildcard.127-0-0-1.sslip.io.pem
#         _wildcard.127-0-0-1.sslip.io-key.pem
```

La limitation: la CA de mkcert est locale. Les autres machines du réseau ne lui feront pas confiance par défaut. Pour un setup solo, c'est très bien. Pour un environnement d'équipe partagé, il faudrait distribuer la CA root, ce qui est essentiellement le même problème opérationnel que traefik.me tentait d'éviter, juste à plus petite échelle.

## La configuration Traefik

Le setup est le même quelle que soit la solution DNS choisie. Traefik a besoin du certificat monté en volume et d'un file provider statique pointant vers un fichier de configuration TLS.

```yaml
# traefik/config/tls.yml
tls:
  certificates:
    - certFile: /certs/cert.pem
      keyFile: /certs/key.pem
  stores:
    default:
      defaultCertificate:
        certFile: /certs/cert.pem
        keyFile: /certs/key.pem
```

La bonne pratique: faire tourner Traefik dans son propre projet Compose, séparé des services qu'il route. Chaque projet de service se connecte à Traefik via un réseau externe partagé. On démarre et arrête les services indépendamment sans toucher au reverse proxy.

On commence par créer le réseau externe une seule fois:

```bash
docker network create traefik-public
```

**`traefik/compose.yml`** - Traefik seul, propriétaire du réseau:

```yaml
services:
  traefik:
    image: traefik:v3
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config:/etc/traefik/config
      - ./certs:/certs
    command:
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker=true
      - --providers.docker.network=traefik-public
      - --providers.file.directory=/etc/traefik/config
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

On copie la sortie de mkcert dans `./certs/`, on renomme en `cert.pem` et `key.pem`, puis:

```bash
docker compose -f traefik/compose.yml up -d
```

Traefik est lancé, il écoute sur le port 80 et 443, et surveille Docker pour les nouveaux containers. Aucune route n'est encore configurée.

**`whoami/compose.yml`** - un service qui rejoint le même réseau:

```yaml
services:
  whoami:
    image: traefik/whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.127-0-0-1.sslip.io`)"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.entrypoints=websecure"
    networks:
      - traefik-public

networks:
  traefik-public:
    external: true
```

```bash
docker compose -f whoami/compose.yml up -d
```

Traefik détecte le nouveau container via le Docker provider, lit ses labels, et ajoute la route. `https://whoami.127-0-0-1.sslip.io` répond immédiatement. Arrêter `whoami` et la route disparaît. Traefik continue de tourner sans s'en apercevoir.

La déclaration `external: true` est la ligne qui porte tout le poids. Sans elle, Compose crée un réseau limité au périmètre du projet: Traefik et `whoami` se retrouvent sur des réseaux différents et ne peuvent pas communiquer, même s'ils tournent tous les deux. Le réseau externe est le bus partagé auquel chaque projet de service doit explicitement adhérer.

Si on préfère les URLs traefik.me, il suffit de remplacer la commande mkcert et le label de host:

```bash
mkcert "*.traefik.me"
```

```yaml
- "traefik.http.routers.whoami.rule=Host(`whoami.traefik.me`)"
```

Le fallback DNS vers 127.0.0.1 gère le reste.

## Ce que l'histoire traefik.me enseigne vraiment

Le modèle de distribution de certificats a toujours été fragile. Une "paire clé publique-clé privée" est une contradiction dans les termes. Chaque révocation était un avertissement que la suivante pourrait être définitive. Finalement, ça l'a été.

La leçon ne se limite pas à traefik.me. Tout service qui apporte de la commodité en supprimant discrètement une frontière de sécurité finira par se heurter à cette frontière. mkcert est le bon outil pour ce problème parce qu'il opère entièrement dans votre propre domaine de confiance: on génère la CA, on l'installe, on émet les certificats. Rien ne dépend de la volonté continue d'un tiers de contourner les règles d'émission de certificats.

sslip.io résout proprement la partie DNS. mkcert résout proprement la partie TLS. Ils se combinent bien. Le setup traefik.me était plus simple, pendant un temps. Jusqu'à ce que ce ne soit plus le cas.
