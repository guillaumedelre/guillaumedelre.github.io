---
title: "HTTPS local avec Traefik : traefik.me est mort, vive sslip.io"
aliases:
  - "/fr/2025/04/17/https-local-avec-traefik-traefik.me-est-mort-vive-sslip.io/"
  - "/2025/04/17/local-https-with-traefik-traefik.me-is-dead-long-live-sslip.io/"
date: 2025-04-17
lastmod: 2026-11-18
tags: [traefik, docker, homelab]
format: experiment
featureimage: "images/traefik-local-https/whoami-https.png"
featureimagecaption: "whoami servi en HTTPS par Traefik sur whoami.127-0-0-1.sslip.io, avec un certificat généré par mkcert"
images: ["/images/traefik-local-https/whoami-https.png"]
description: "Le certificat wildcard de traefik.me a été révoqué en 2025. Voici comment le remplacer avec sslip.io, mkcert et une configuration Traefik locale."
---

{{< lead >}}
La configuration semblait parfaite. Pointer `*.traefik.me` sur 127.0.0.1, télécharger un certificat wildcard depuis le même domaine, le déposer dans Traefik, et chaque service local obtient une URL HTTPS propre sans IP dans la barre d'adresse. Pas de limites de débit Let's Encrypt, pas de `mkcert` à expliquer aux collègues, pas d'avertissements de certificat auto-signé à contourner. Rien que `https://myapp.traefik.me` et un cadenas vert.
{{< /lead >}}

Puis en mars 2025, Let's Encrypt a révoqué le certificat wildcard de traefik.me, et il a disparu pour de bon.

## Ce que traefik.me vendait

traefik.me est un résolveur DNS wildcard. Tapez `anything.traefik.me` et ça pointe vers 127.0.0.1. Tapez `anything.10.0.0.1.traefik.me` et ça pointe vers 10.0.0.1. Aucun compte, rien à configurer ni à maintenir. La partie DNS fonctionne toujours bien, soit dit en passant.

Le certificat était le bonus : un certificat wildcard pour `*.traefik.me` que pyrou, le mainteneur, avait généré avec Let's Encrypt et distribué sur `https://traefik.me/cert.pem` et `https://traefik.me/privkey.pem`. C'était pratique précisément parce que c'était partagé : télécharger, déposer dans Traefik, terminé.

Partager une clé privée, c'est ce qui l'a tué.

Les Baseline Requirements du CA/Browser Forum, section 9.6.3, imposent à chaque souscripteur de s'engager à « keep confidential, and properly protect at all times the Private Key », autrement dit à garder sa clé privée confidentielle et protégée en permanence. La distribuer à quiconque visite une URL, c'est exactement le contraire. Le 24 mars 2025, un membre de l'équipe Let's Encrypt [l'a annoncé sur le dépôt du projet][le-post] : la clé divulguée a été bloquée, les certificats associés révoqués, avec l'avertissement qu'un souscripteur qui ignore les règles verrait le domaine bloqué. Pyrou [a confirmé la situation][pyrou-reply] et recommandé mkcert comme alternative. Le projet survivra uniquement en tant que résolveur DNS.

Ce n'était pas la première fois. En 2022, pyrou expliquait dans [une issue][issue-1] que le certificat avait déjà été révoqué deux fois, et prévenait : si quelqu'un s'acharnait à le faire révoquer dès sa publication, il arrêterait de distribuer la clé privée. Cette fois, c'était Let's Encrypt lui-même, et c'était la dernière.

## sslip.io fait la même chose, différemment

[sslip.io][sslip] est aussi un résolveur DNS wildcard. Comme avec traefik.me, l'IP est dans le hostname, avec des points ou des tirets ; contrairement à traefik.me, il n'y a pas de repli quand elle est absente. `10-0-0-1.sslip.io` pointe vers `10.0.0.1`. `myapp.192-168-1-10.sslip.io` pointe vers `192.168.1.10`. IPv6 fonctionne aussi.

L'infrastructure derrière sslip.io est aussi plus visible : trois serveurs de noms à Singapour, aux États-Unis et en Pologne, qui traitent plus de 20 000 requêtes par seconde selon le site du projet. Plus de 1 200 étoiles sur GitHub et une maintenance active, sous licence Apache 2.0 (chiffres de novembre 2026).

En mettant de côté l'histoire des certificats, la comparaison est assez directe :

| | traefik.me | sslip.io |
|---|---|---|
| DNS wildcard | oui | oui |
| Fallback vers 127.0.0.1 | oui | non |
| Certificat wildcard | ~~oui~~ révoqué | non |
| Infrastructure | opaque | documentée |
| Activité du projet | faible | active |

Le seul avantage restant de traefik.me est le repli vers 127.0.0.1 : des URLs sans segment IP. C'est important si vous tenez à `myapp.traefik.me` plutôt que `myapp.127-0-0-1.sslip.io`. Savoir si cette différence vaut l'incertitude sur l'infrastructure, c'est une discussion qui tourne court.

## mkcert comble le vide

[mkcert][mkcert] crée une autorité de certification locale, l'installe dans le magasin de certificats du système et dans les navigateurs qu'il trouve, puis émet des certificats signés par cette CA. Les navigateurs voient une chaîne de confiance valide et ne protestent plus.

```bash
mkcert -install
```

C'est la configuration unique. Ensuite, générer un certificat se fait en une commande :

```bash
mkcert "*.127-0-0-1.sslip.io"
# produit _wildcard.127-0-0-1.sslip.io.pem
#         _wildcard.127-0-0-1.sslip.io-key.pem
```

{{< figure src="images/traefik-local-https/verifie-par-mkcert.png" alt="Onglet Sécurité de Firefox pour whoami.127-0-0-1.sslip.io : vérifiée par mkcert development CA, connexion chiffrée en TLS 1.3" caption="Le navigateur fait confiance au certificat parce qu'il connaît l'autorité locale : « Vérifiée par : mkcert development CA »." >}}

La limitation : la CA de mkcert est locale. Les autres machines du réseau ne lui feront pas confiance par défaut. Pour une configuration solo, c'est très bien. Pour un environnement d'équipe partagé, il faudrait distribuer la CA racine, ce qui revient au même problème opérationnel que traefik.me tentait d'éviter, à plus petite échelle.

## La configuration Traefik

La configuration est la même quelle que soit la solution DNS choisie. Traefik a besoin du certificat monté en volume et d'un file provider, déclaré dans la configuration statique, qui charge un fichier de configuration TLS dynamique.

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

La bonne pratique : faire tourner Traefik dans son propre projet Compose, séparé des services qu'il route. Chaque projet de service se connecte à Traefik via un réseau externe partagé. Vous démarrez et arrêtez les services indépendamment sans toucher au reverse proxy.

Commencez par créer le réseau externe une seule fois :

```bash
docker network create traefik-public
```

**`traefik/compose.yml`** - Traefik seul, rattaché au réseau partagé :

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

Copiez la sortie de mkcert dans `./certs/`, renommez les fichiers en `cert.pem` et `key.pem`, puis :

```bash
docker compose -f traefik/compose.yml up -d
```

Traefik est lancé, il écoute sur les ports 80 et 443, et surveille Docker pour les nouveaux containers. Aucune route n'est encore configurée.

**`whoami/compose.yml`** - un service qui rejoint le même réseau :

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

Traefik détecte le nouveau container via le Docker provider, lit ses labels, et ajoute la route. `https://whoami.127-0-0-1.sslip.io` répond immédiatement. Arrêtez `whoami` : la route disparaît, et Traefik continue de tourner sans s'en apercevoir.

La déclaration `external: true` est la ligne qui fait tout le travail. Sans elle, Compose crée un réseau limité au périmètre du projet : Traefik et `whoami` se retrouvent sur des réseaux différents et ne peuvent pas communiquer, même s'ils tournent tous les deux. Le réseau externe est le bus partagé auquel chaque projet de service doit explicitement adhérer.

Si vous préférez les URLs traefik.me, il suffit de remplacer la commande mkcert et le label de host :

{{< tabs >}}
{{< tab label="sslip.io" >}}
```bash
mkcert "*.127-0-0-1.sslip.io"
```

```yaml
- "traefik.http.routers.whoami.rule=Host(`whoami.127-0-0-1.sslip.io`)"
```
{{< /tab >}}
{{< tab label="traefik.me" >}}
```bash
mkcert "*.traefik.me"
```

```yaml
- "traefik.http.routers.whoami.rule=Host(`whoami.traefik.me`)"
```
{{< /tab >}}
{{< /tabs >}}

Le repli DNS vers 127.0.0.1 gère le reste.

## Ce que l'histoire traefik.me enseigne vraiment

Le modèle de distribution de certificats a toujours été fragile. Une « paire clé publique-clé privée » est une contradiction dans les termes. Chaque révocation était un avertissement que la suivante pourrait être définitive. Finalement, ça l'a été.

La leçon ne se limite pas à traefik.me. Tout service qui apporte de la commodité en supprimant discrètement une frontière de sécurité finira par se heurter à cette frontière. mkcert est le bon outil pour ce problème parce qu'il opère entièrement dans votre propre domaine de confiance : vous générez la CA, vous l'installez, vous émettez les certificats. Rien ne dépend de la volonté continue d'un tiers de contourner les règles d'émission de certificats.

sslip.io résout proprement la partie DNS. mkcert résout proprement la partie TLS. Ils se combinent bien. La configuration traefik.me était plus simple, pendant un temps. Jusqu'à ce que ce ne soit plus le cas.

[le-post]: https://github.com/pyrou/traefik.me/discussions/14
[pyrou-reply]: https://github.com/pyrou/traefik.me/discussions/15
[issue-1]: https://github.com/pyrou/traefik.me/issues/1
[sslip]: https://sslip.io
[mkcert]: https://github.com/FiloSottile/mkcert
