---
title: "Pourquoi je n'ai jamais installé mkcert dans mon homelab"
date: 2026-11-19
tags: [traefik, homelab, docker]
format: experiment
description: "J'avais prévu mkcert pour le HTTPS de mon homelab. J'ai pris un vrai domaine, un certificat Let's Encrypt et Pi-hole, sans ouvrir de port."
---

{{< lead >}}
La dernière fois que j'ai parlé du HTTPS de mon homelab, je finissais sur une liste de quatre étapes pour le remettre avec mkcert. Je n'en ai fait aucune : la quatrième, installer l'autorité sur chaque appareil qui consulte le homelab, m'a fait changer de plan avant même de commencer. À la place, j'ai pris un nom de domaine, gratuit la première année puis 8 euros par an, et aucun appareil de la maison n'a eu besoin d'être touché.
{{< /lead >}}

Ce qui suit raconte pourquoi j'ai abandonné mkcert, comment un homelab fermé à Internet obtient quand même un vrai certificat Let's Encrypt, et les deux pièges tendus par le DNS que j'avais installé juste avant.

## La quatrième étape

Dans [l'article sur la fin de traefik.me]({{< relref "2025-04-17-traefik-local-https-mkcert-sslip.md" >}}), mkcert était la solution propre : une autorité de certification à moi, une clé que personne d'autre ne détient. Les trois premières étapes tenaient dans un dossier. La quatrième tenait dans la maison.

J'ai compté : trois ordinateurs, des téléphones, des tablettes, dont celles des enfants. Sur un ordinateur, importer `rootCA.pem` prend une minute. Sur un téléphone Android, il faut fouiller les paramètres de sécurité, et certaines applications ignorent de toute façon les autorités ajoutées par l'utilisateur. Sur iOS, il faut installer un profil, puis activer la confiance totale dans un autre menu. Et chaque nouvel appareil qui arrive dans la maison recommence la procédure, ou affiche un avertissement de sécurité que les enfants apprendront à ignorer. C'est le pire résultat possible : un HTTPS qui éduque à cliquer sur « Accepter le risque ».

Le problème n'était donc pas le certificat, mais la confiance. mkcert la crée, et il faut ensuite la distribuer. Un certificat Let's Encrypt, lui, est déjà accepté par tous les appareils du monde. Il lui manque une seule chose : un nom de domaine qui m'appartient.

## Un vrai domaine pour des adresses privées

Le réflexe, quand on pense « Let's Encrypt », c'est le challenge HTTP-01 : Let's Encrypt appelle `http://monservice.example.com/.well-known/...` et vérifie la réponse. Il faut donc que la machine soit joignable depuis Internet. Mon homelab ne l'est pas et ne le sera pas : plusieurs services n'ont aucune authentification, et c'est un choix qui ne tient que sur le réseau local.

Le challenge DNS-01 contourne le problème. Pour prouver que le domaine m'appartient, Traefik crée un enregistrement TXT `_acme-challenge` dans la zone, Let's Encrypt le lit, et le certificat est émis. La machine n'a jamais besoin d'être joignable. C'est aussi le seul challenge qui donne droit à un certificat wildcard, et c'est ce qui m'intéressait : un seul certificat pour `*.example.com`, et chaque nouvelle stack est en HTTPS sans rien demander.

J'ai pris un `.fr` chez OVH, avec une offre du moment : engagement de trois ans, première année offerte, puis 8 euros par an. Traefik sait piloter l'API d'OVH grâce à [lego][lego], la bibliothèque ACME qu'il embarque. La clé d'API que j'ai créée n'a de droits que sur la zone DNS de ce domaine, rien d'autre sur le compte. Côté Traefik, tout se règle dans la configuration statique :

```yaml
entryPoints:
    websecure:
        address: :443
        http3: {}
        http:
            tls:
                certResolver: letsencrypt
                domains:
                    - main: example.com
                      sans:
                          - "*.example.com"

certificatesResolvers:
    letsencrypt:
        acme:
            storage: /letsencrypt/acme.json
            keyType: EC256
            dnsChallenge:
                provider: ovh
```

Les quatre variables `OVH_ENDPOINT`, `OVH_APPLICATION_KEY`, `OVH_APPLICATION_SECRET` et `OVH_CONSUMER_KEY` arrivent par le `.env` de la stack Traefik. Le détail qui m'a plu : le TLS est déclaré sur le point d'entrée, pas sur chaque service. Les routeurs n'ont plus besoin du label `tls`, il suffit qu'ils écoutent sur `websecure`. Le port 80 ne sert plus qu'à rediriger vers le HTTPS.

Restait la question qui fait hésiter : quelle adresse mettre dans la zone publique ? J'ai mis l'adresse privée, `10.0.0.10`, pour l'apex et pour `*`. Depuis l'extérieur, ce nom pointe vers une adresse qui ne mène nulle part. Depuis la maison, il mène au homelab. Et puisque le certificat est un wildcard, les journaux publics de Certificate Transparency ne contiennent que `*.example.com` : la liste de mes services n'y apparaît pas.

## Le DNS installé pour autre chose

Peu avant, j'avais installé [Pi-hole][pihole] pour une tout autre raison : protéger le réseau familial. Listes de blocage pour la publicité, les contenus pour adultes et les jeux d'argent, appliquées à toute la maison. Le routeur annonce Pi-hole comme DNS, et chaque appareil en profite sans réglage.

C'est lui qui m'a coûté le plus de temps, deux fois.

### Le port 53 était déjà pris

Première tentative : publier le port 53, comme dans tous les exemples.

```yaml
ports:
    - "53:53/tcp"
    - "53:53/udp"
```

Le conteneur refuse de démarrer, avec un `address already in use`. Sur Ubuntu, `systemd-resolved` écoute déjà sur le port 53, sur `127.0.0.53` et `127.0.0.54`. La plupart des tutoriels conseillent alors de désactiver son écouteur, ce qui revient à toucher à la résolution de noms de la machine elle-même pour faire plaisir à un conteneur. Je n'aime pas cette idée.

La bonne correction est plus petite : publier le port uniquement sur l'adresse du réseau local, qui, elle, est libre.

```yaml
ports:
    - "${IP}:53:53/tcp"
    - "${IP}:53:53/udp"
```

`systemd-resolved` garde ses adresses de bouclage, Pi-hole prend l'adresse que le reste de la maison interroge, et chacun reste chez soi.

### Le challenge invisible

Pour que le homelab réponde même quand la connexion Internet tombe, j'ai demandé à Pi-hole de répondre lui-même pour tout le domaine, avec une ligne dnsmasq :

```text
address=/example.com/10.0.0.10
```

Les services marchaient, la résolution était instantanée, et le certificat n'arrivait pas. Traefik créait bien l'enregistrement TXT chez OVH, puis attendait de le voir apparaître avant de prévenir Let's Encrypt. Il ne le voyait jamais.

Il a fallu un moment pour comprendre que Traefik posait la question à Pi-hole, comme tous les conteneurs de la machine. Et Pi-hole, qui se considère maintenant comme la source de vérité pour `example.com`, ne transmet plus aucune question sur ce domaine à OVH. L'enregistrement existait bien dans la zone publique, mais restait invisible depuis le homelab. Le DNS que j'avais installé pour simplifier la vie de la maison masquait exactement ce que Traefik devait vérifier.

La correction tient en deux lignes dans la configuration du challenge : imposer des résolveurs publics pour la vérification.

```yaml
dnsChallenge:
    provider: ovh
    resolvers:
        - "1.1.1.1:53"
        - "9.9.9.9:53"
```

Le reste de la machine continue de passer par Pi-hole. Seule la vérification du challenge le contourne. Et les enregistrements publics pointant vers `10.0.0.10` gardent une utilité : un appareil qui contourne Pi-hole, avec un DNS privé configuré sur le téléphone par exemple, trouve quand même le homelab.

## Un domaine qui n'envoie pas de mail

Un domaine sans configuration mail est une cible facile : n'importe qui peut envoyer un message en se faisant passer pour `quelquechose@example.com`, et rien ne dit aux serveurs de destination de le refuser. Le mien n'enverra jamais de mail, alors la zone le dit explicitement : un MX nul, un SPF `v=spf1 -all`, une politique DMARC `reject`. J'ai ajouté un enregistrement CAA qui n'autorise que Let's Encrypt à émettre des certificats pour le domaine, et activé DNSSEC. Rien de tout ça n'est nécessaire pour avoir le cadenas, mais un domaine public, même pointé sur une adresse privée, reste un domaine public.

## Les anciennes adresses

Les anciens noms en `*.10.0.0.10.sslip.io` étaient partout : favoris, Home Assistant, applications mobiles. Plutôt que de les casser, le point d'entrée `web` les redirige vers le nouveau domaine, avec une règle `HostRegexp` et un middleware `redirectRegex` dans la configuration dynamique de Traefik.

Ça marche pour un navigateur. Ça ne marche pas pour un client qui envoie des `POST` : une redirection 301 transforme souvent la requête en `GET`, ou ne la suit pas du tout. Les hooks qui alimentent [claude-agents-board]({{< relref "2026-11-17-claude-agents-board.md" >}}) depuis mes trois machines envoient des `POST` : il a fallu les réinstaller avec la nouvelle URL. La redirection est une politesse pour les humains, pas une migration.

## Ce que je garde

Je pensais que le vrai HTTPS dans un homelab fermé imposait de choisir entre une autorité maison à distribuer partout et une machine exposée sur Internet. Le challenge DNS-01 supprime ce dilemme pour le prix d'un nom de domaine, et la clé d'API limitée à une zone rend le compromis acceptable. mkcert reste le bon outil sur un poste de développement, où il n'y a qu'une machine à convaincre. Pour une maison, le problème à régler est la distribution de la confiance, et Let's Encrypt l'a déjà réglé.

Ce qui m'a surpris, c'est que le temps perdu ne venait ni de Traefik ni de Let's Encrypt, mais de Pi-hole, installé pour une raison sans rapport. Un DNS local qui répond pour un domaine public change ce que voit chaque conteneur de la machine, y compris celui qui doit vérifier ce domaine. Si vous faites la même chose, configurez les résolveurs du challenge avant d'en avoir besoin.

[lego]: https://go-acme.github.io/lego/dns/ovh/
[pihole]: https://github.com/pi-hole/pi-hole
