---
title: "HTTPS dans un homelab : domaine, challenge DNS et Pi-hole"
date: 2026-11-19
tags: [traefik, homelab, docker]
format: guide
featureimage: "images/homelab-https/traefik-routers.png"
featureimagecaption: "Les routeurs de mon Traefik : chaque service écoute sur websecure avec le bouclier TLS, le point d'entrée web ne fait que rediriger. Les noms d'hôte sont floutés."
images: ["/images/homelab-https/traefik-routers.png"]
description: "Guide pas à pas : un certificat Let's Encrypt wildcard pour un homelab fermé à Internet, avec un domaine, le challenge DNS-01 et Pi-hole."
---

{{< lead >}}
Ce guide met en place un vrai HTTPS dans un homelab qui n'est pas exposé sur Internet. À la fin, chaque service répond sur `https://<service>.example.com` avec un certificat Let's Encrypt que tous les appareils de la maison acceptent déjà, sans rien installer sur aucun d'eux et sans ouvrir le moindre port.
{{< /lead >}}

Dans [l'article sur la fin de traefik.me]({{< relref "2025-04-17-traefik-local-https-mkcert-sslip.md" >}}), je prévoyais de remettre le HTTPS avec mkcert. Je ne l'ai jamais fait : il aurait fallu installer mon autorité de certification sur chaque ordinateur, chaque téléphone et chaque tablette de la maison, puis recommencer à chaque nouvel appareil. Un certificat Let's Encrypt est déjà accepté partout. Il lui manquait un nom de domaine, et j'en ai pris un chez OVH avec une offre du moment : engagement de trois ans, première année offerte, puis 8 euros par an.

Le guide s'adresse à quelqu'un qui a déjà un homelab derrière Traefik. Les exemples viennent de ma configuration, qui suit [les principes décrits dans cet article]({{< relref "2026-05-17-homelab-docker-compose-traefik.md" >}}) ; seuls le domaine (remplacé par `example.com`) et les secrets ont été changés.

{{< alert icon="list-check" >}}
**Prérequis**

- Traefik v3 dans Docker, avec un réseau partagé et des services routés par labels (testé avec Traefik 3.7).
- Une machine avec une adresse fixe sur le réseau local, ici `10.0.0.10`.
- Un nom de domaine chez un registraire dont [lego](https://go-acme.github.io/lego/dns/), la bibliothèque ACME de Traefik, sait piloter l'API. J'utilise OVH.
- Facultatif : Pi-hole, pour résoudre les noms localement (étape 6).
{{< /alert >}}

## Pourquoi le challenge DNS-01

Let's Encrypt doit vérifier que le domaine vous appartient. Le challenge le plus connu, HTTP-01, appelle une URL sur votre serveur depuis Internet : il faut donc ouvrir le port 80 vers le homelab. Le mien a plusieurs interfaces sans authentification, et ce choix ne tient que s'il reste fermé. Avec DNS-01, Traefik dépose un enregistrement TXT dans la zone du domaine par l'API du registraire, Let's Encrypt le lit, et la machine n'a jamais besoin d'être joignable. C'est aussi le seul challenge qui donne droit à un certificat wildcard : un seul certificat pour `*.example.com`, et chaque nouveau service est en HTTPS sans rien demander.

## 1. Pointer le domaine vers l'adresse privée

Dans la zone DNS du domaine, créez deux enregistrements A vers l'adresse locale du homelab :

```text
@    A    10.0.0.10
*    A    10.0.0.10
```

Oui, une adresse privée dans une zone publique. Depuis l'extérieur, ces noms mènent à une adresse que personne ne peut joindre ; depuis la maison, ils mènent au homelab. Le wildcard a un autre avantage : les journaux publics de Certificate Transparency, où chaque certificat émis est inscrit, ne contiennent que `*.example.com`. La liste de vos services n'y apparaît pas.

**Vérifier :**

```bash
dig +short dozzle.example.com @1.1.1.1
# 10.0.0.10
```

**Si la réponse est vide** alors que l'enregistrement existe, la zone n'a pas encore été publiée ; chez OVH, une modification peut prendre quelques minutes. Certains routeurs et résolveurs filtrent aussi les réponses publiques qui contiennent une adresse privée (protection contre le « DNS rebinding ») : interroger `1.1.1.1` directement, comme ci-dessus, permet de faire la différence. L'étape 6 règle ce cas pour la maison.

## 2. Créer une clé d'API limitée à la zone

Traefik va écrire dans votre zone DNS. Donnez-lui le moins de droits possible : chez OVH, la [documentation de lego](https://go-acme.github.io/lego/dns/ovh/) demande `POST` et `DELETE` sur `/domain/zone/*`. Je les ai restreints à la seule zone du domaine :

```text
POST    /domain/zone/example.com/*
DELETE  /domain/zone/example.com/*
```

Avec cette clé, quelqu'un qui la volerait pourrait modifier la zone, mais ni commander un service, ni toucher au reste du compte. Rangez les quatre valeurs obtenues dans le `.env` de la stack Traefik, qui n'est pas versionné :

```dotenv
OVH_ENDPOINT=ovh-eu
OVH_APPLICATION_KEY=...
OVH_APPLICATION_SECRET=...
OVH_CONSUMER_KEY=...
```

Et transmettez-les au conteneur dans son `compose.yaml` :

```yaml
environment:
    OVH_ENDPOINT: ${OVH_ENDPOINT}
    OVH_APPLICATION_KEY: ${OVH_APPLICATION_KEY}
    OVH_APPLICATION_SECRET: ${OVH_APPLICATION_SECRET}
    OVH_CONSUMER_KEY: ${OVH_CONSUMER_KEY}
```

Avec un autre registraire, seuls les noms des variables changent : chaque fournisseur a sa page dans la documentation de lego.

## 3. Demander le certificat avec Traefik

Dans la configuration statique de Traefik, deux blocs : le point d'entrée `websecure`, qui porte le certificat, et le résolveur qui l'obtient.

```yaml
entryPoints:
    web:
        address: :80
    websecure:
        address: :443
        http3: {}
        http:
            # TLS sur tous les routeurs de ce point d'entrée
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
                # Résolveurs publics pour vérifier le challenge (voir l'étape 6)
                resolvers:
                    - "1.1.1.1:53"
                    - "9.9.9.9:53"
```

Déclarer le TLS sur le point d'entrée plutôt que sur chaque routeur, c'est ce qui rend la suite simple : un service n'a plus besoin du label `tls`, il lui suffit d'écouter sur `websecure`. Côté `compose.yaml`, publiez le port 443 en TCP et en UDP (le second pour HTTP/3) et montez un dossier pour `acme.json` :

```yaml
ports:
    - "80:80"
    - "443:443"
    - "443:443/udp"
volumes:
    - ./docker/letsencrypt:/letsencrypt
```

{{< alert >}}
Pendant les essais, ajoutez `caServer: https://acme-staging-v02.api.letsencrypt.org/directory` sous `acme`. Les certificats de test ne sont pas reconnus par les navigateurs, mais ils ne consomment pas les limites d'émission de Let's Encrypt. Retirez la ligne et déplacez `acme.json` pour obtenir le vrai.
{{< /alert >}}

**Vérifier**, une à deux minutes après le redémarrage de Traefik :

```bash
echo | openssl s_client -connect 10.0.0.10:443 -servername dozzle.example.com 2>/dev/null \
  | openssl x509 -noout -issuer -ext subjectAltName -enddate
```

L'émetteur doit contenir `O = Let's Encrypt`, et les noms alternatifs `*.example.com` et `example.com`. Le tableau de bord de Traefik affiche aussi le certificat dans sa page « Certificates », avec sa date d'expiration. Traefik le renouvelle seul.

**Si l'émetteur est `TRAEFIK DEFAULT CERT`**, le certificat n'a pas été obtenu : les logs de Traefik disent pourquoi (`docker compose logs traefik`). Les deux causes les plus courantes sont une clé d'API sans les bons droits et un fichier `acme.json` créé à la main avec des permissions trop larges : Traefik exige `600`. Le plus simple est de le laisser créer le fichier lui-même.

## 4. Passer les services sur websecure

Chaque service garde son label de routage habituel, avec le point d'entrée `websecure`. Voici les labels de Dozzle chez moi :

```yaml
labels:
    traefik.enable: true
    traefik.http.routers.dozzle.entrypoints: websecure
    traefik.http.routers.dozzle.rule: Host(`${COMPOSE_PROJECT_NAME}.${DOMAIN}`)
    traefik.http.services.dozzle.loadbalancer.server.port: 8080

    homepage.href: https://${COMPOSE_PROJECT_NAME}.${DOMAIN}
```

avec `DOMAIN=example.com` dans le `.env` de la stack. Pensez aux liens : une carte Homepage ou une icône servie par l'application elle-même en `http://` provoque un avertissement de contenu mixte une fois le tableau de bord en HTTPS.

**Vérifier**, depuis une machine du réseau :

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://dozzle.example.com
# 200
```

Une erreur de certificat ici, alors que l'étape 3 est validée, veut dire que la règle `Host` ne correspond pas au nom demandé : Traefik répond alors avec son certificat par défaut.

## 5. Rediriger le HTTP et les anciennes adresses

Le port 80 ne sert plus qu'à rediriger. Je le fais avec des routeurs dans la configuration dynamique de Traefik (provider `file`), parce que j'avais aussi d'anciennes adresses en `sslip.io` à rattraper :

```yaml
http:
    routers:
        http-to-https:
            entryPoints: [web]
            rule: PathPrefix(`/`)
            priority: 1
            middlewares: [to-https]
            service: noop@internal
        legacy-sslip:
            entryPoints: [web]
            rule: HostRegexp(`^[a-z0-9-]+\.10\.0\.0\.10\.sslip\.io$`)
            priority: 10
            middlewares: [legacy-sslip]
            service: noop@internal

    middlewares:
        to-https:
            redirectScheme:
                scheme: https
                permanent: true
        legacy-sslip:
            redirectRegex:
                regex: ^https?://([a-z0-9-]+)\.10\.0\.0\.10\.sslip\.io(.*)
                replacement: https://${1}.example.com${2}
                permanent: true
```

Sans anciennes adresses, la redirection intégrée au point d'entrée (`entryPoints.web.http.redirections`) suffit.

**Vérifier :**

```bash
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://dozzle.10.0.0.10.sslip.io
# 301 https://dozzle.example.com/
```

**Le piège :** une redirection convient à un navigateur, pas forcément aux scripts. Traefik répond `301` aux `GET` et `308` aux autres méthodes, ce qui conserve un `POST`, mais encore faut-il que le client suive la redirection : `curl` sans `-L` ne la suit pas. Les hooks qui alimentent [claude-agents-board]({{< relref "2026-11-17-claude-agents-board.md" >}}) depuis mes trois machines envoient des `POST` : il a fallu les réinstaller avec la nouvelle URL. Faites la liste de ce qui appelle vos services sans navigateur (webhooks, scripts, applications mobiles, `internal_url` de Home Assistant) avant de basculer.

## 6. Répondre localement avec Pi-hole

Cette étape est facultative : les enregistrements publics de l'étape 1 suffisent tant qu'Internet fonctionne. J'avais installé [Pi-hole][pihole] juste avant, pour protéger le réseau familial avec des listes de blocage, et je m'en sers aussi pour que le homelab réponde même quand la connexion tombe. C'est l'étape qui m'a coûté le plus de temps, deux fois.

L'extrait utile du `compose.yaml` de Pi-hole (v6) :

```yaml
ports:
    - "${IP}:53:53/tcp"
    - "${IP}:53:53/udp"
environment:
    # Sur un réseau bridge Docker, les requêtes arrivent de la passerelle du bridge
    FTLCONF_dns_listeningMode: ALL
    # Tout le domaine répond localement
    FTLCONF_misc_dnsmasq_lines: address=/example.com/10.0.0.10
```

Déclarez ensuite Pi-hole comme serveur DNS dans votre routeur, pour que chaque appareil l'utilise sans réglage.

**Vérifier :**

```bash
dig +short dozzle.example.com @10.0.0.10
# 10.0.0.10
```

### Symptôme : le conteneur refuse de démarrer

Avec la publication habituelle `"53:53/udp"`, Docker échoue sur `address already in use`. **Cause :** sur Ubuntu, `systemd-resolved` écoute déjà sur le port 53, sur `127.0.0.53` et `127.0.0.54`. La plupart des tutoriels conseillent de désactiver cet écouteur, ce qui revient à modifier la résolution de noms de la machine pour faire plaisir à un conteneur. **Correction :** publier le port sur la seule adresse du réseau local, comme dans l'extrait ci-dessus. `systemd-resolved` garde ses adresses de bouclage, Pi-hole prend celle que la maison interroge.

### Symptôme : le certificat n'arrive plus

Les services répondaient, la résolution était instantanée, et aucun certificat n'était émis. Traefik créait bien l'enregistrement TXT chez OVH, puis attendait de le voir apparaître avant de prévenir Let's Encrypt, et ne le voyait jamais. **Cause :** Traefik résout les noms comme tous les conteneurs de la machine, donc par Pi-hole. Et Pi-hole, avec sa ligne `address=/example.com/...`, répond lui-même pour tout le domaine et ne transmet plus aucune question à OVH. L'enregistrement existait dans la zone publique, mais restait invisible depuis le homelab. **Correction :** les `resolvers` de l'étape 3, qui imposent des résolveurs publics pour la seule vérification du challenge. Si vous ajoutez Pi-hole après coup, c'est la ligne à ne pas oublier.

Les enregistrements publics de l'étape 1 gardent leur utilité : un appareil qui contourne Pi-hole, un téléphone avec un DNS privé par exemple, trouve quand même le homelab.

## 7. Fermer la zone au mail usurpé

Un domaine sans configuration mail est une cible facile : n'importe qui peut envoyer un message au nom de `quelquechose@example.com`, et rien ne dit aux serveurs de destination de le refuser. Le mien n'enverra jamais de mail, alors la zone le dit explicitement. Un enregistrement CAA limite en plus l'émission de certificats à Let's Encrypt :

```text
@             MX     0 .
@             TXT    "v=spf1 -all"
_dmarc        TXT    "v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s"
*._domainkey  TXT    "v=DKIM1; p="
@             CAA    0 issue "letsencrypt.org"
```

Le MX nul déclare que le domaine ne reçoit pas de mail, le SPF qu'aucun serveur n'est autorisé à en envoyer, le DKIM vide qu'aucune clé de signature n'est valide, et DMARC demande de rejeter tout ce qui prétend le contraire. J'ai aussi activé DNSSEC dans l'espace client OVH. Rien de tout ça n'est nécessaire pour obtenir le cadenas, mais un domaine pointé sur une adresse privée reste un domaine public.

**Vérifier :**

```bash
dig +short MX example.com @1.1.1.1          # 0 .
dig +short TXT _dmarc.example.com @1.1.1.1
dig +short CAA example.com @1.1.1.1
dig +dnssec +short DS example.com @1.1.1.1  # non vide une fois DNSSEC actif
```

## Ce que ce guide ne couvre pas

Le HTTPS chiffre les échanges, il n'authentifie personne : un service sans mot de passe reste ouvert à tout appareil du réseau local, cadenas ou pas. Le guide ne rend rien accessible depuis Internet non plus ; pour ça, il faudrait un tunnel ou un VPN, et une vraie couche d'authentification devant les services. Il ne couvre pas d'autre registraire qu'OVH, mais seule l'étape 2 change : la [liste des fournisseurs de lego](https://go-acme.github.io/lego/dns/) donne les variables de chacun. Enfin, mkcert reste le bon outil sur un poste de développement, où il n'y a qu'une machine à convaincre.

[pihole]: https://github.com/pi-hole/pi-hole
