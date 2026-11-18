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
description: "J'utilisais le certificat partagé de traefik.me jusqu'à sa révocation en 2025. sslip.io a remplacé le DNS, mkcert remplacera le HTTPS."
---

{{< lead >}}
Le dossier qui contient mon homelab s'appelle toujours `traefik.me`. C'est le dernier vestige d'une époque où chacun de mes services avait une adresse en HTTPS sans que j'aie rien à faire. Depuis mars 2025, ils sont tous en HTTP, et mon Traefik publie un port 443 que plus personne n'écoute.
{{< /lead >}}

Cet article raconte ce qui a cassé, ce que j'ai remplacé tout de suite, et ce qu'il faut pour remettre le HTTPS, testé de bout en bout pour l'occasion.

## Un certificat que tout le monde pouvait télécharger

[traefik.me][traefik-me] fait deux choses. La première, c'est un DNS joker : `anything.traefik.me` pointe vers `127.0.0.1`, et `anything.10.0.0.1.traefik.me` vers `10.0.0.1`. Aucun compte, rien à configurer. Cette partie fonctionne toujours.

La seconde, c'était le certificat. Pyrou, le mainteneur, générait avec Let's Encrypt un certificat wildcard pour `*.traefik.me` et le publiait, clé privée comprise, sur `https://traefik.me/cert.pem` et `https://traefik.me/privkey.pem`. On téléchargeait les deux fichiers, on les déposait dans Traefik, et tous les services avaient un cadenas, y compris ceux qui ne tournaient que sur le réseau de la maison. C'est ce que je faisais.

Le problème tient dans les fichiers eux-mêmes : une clé privée que n'importe qui peut télécharger n'a plus rien de privé.

## La révocation

Les règles qui s'imposent à toutes les autorités de certification (les Baseline Requirements du CA/Browser Forum, section 9.6.3) obligent chaque titulaire d'un certificat à garder sa clé privée confidentielle. La publier sur une URL, c'est exactement l'inverse.

Ce n'était pas une surprise, et la fin s'est jouée en deux jours :

{{< timeline >}}
{{< timelineItem icon="triangle-exclamation" header="Deux révocations déjà" badge="2022" md=true >}}
Dans [une issue du projet](https://github.com/pyrou/traefik.me/issues/1), pyrou indique que le certificat a déjà été révoqué deux fois. Il prévient : si quelqu'un s'acharne à le faire révoquer dès sa publication, il arrêtera de distribuer la clé privée.
{{< /timelineItem >}}
{{< timelineItem icon="lock" header="Let's Encrypt révoque le certificat" badge="24 mars 2025" md=true >}}
Un membre de l'équipe Let's Encrypt [l'écrit sur le dépôt du projet](https://github.com/pyrou/traefik.me/discussions/14) : la clé publiée est bloquée, les certificats associés révoqués, et le domaine pourrait être interdit d'émission si ça continue.
{{< /timelineItem >}}
{{< timelineItem icon="xmark" header="Fin du certificat partagé" badge="25 mars 2025" md=true >}}
Pyrou [prend acte](https://github.com/pyrou/traefik.me/discussions/15) : il arrête de distribuer le certificat, recommande mkcert et garde traefik.me comme simple service DNS.
{{< /timelineItem >}}
{{< /timeline >}}

## Ce que j'ai remplacé tout de suite

Un homelab sur traefik.me dépend de lui pour deux choses : les noms et le certificat. Les noms se remplacent facilement, parce que d'autres services font la même chose.

J'ai pris [sslip.io][sslip]. Le principe est identique : l'adresse IP est écrite dans le nom, avec des points ou des tirets. `dozzle.10.0.0.10.sslip.io` pointe vers `10.0.0.10`, et `myapp.192-168-1-10.sslip.io` vers `192.168.1.10`. Dans mon homelab, chaque stack lit l'adresse et le domaine dans son propre `.env` ([le principe 3 de cet article]({{< relref "2026-05-17-homelab-docker-compose-traefik.md" >}}) explique comment). Changer de domaine revient à modifier une ligne par stack.

sslip.io a aussi l'avantage d'être plus transparent sur son fonctionnement : trois serveurs de noms à Singapour, aux États-Unis et en Pologne, plus de 20 000 requêtes par seconde selon le site du projet, plus de 1 200 étoiles sur GitHub et une maintenance active sous licence Apache 2.0 (chiffres de novembre 2026). Les deux services diffèrent sur trois points :

| | traefik.me | sslip.io |
|---|---|---|
| Nom sans adresse IP (`myapp.traefik.me`) | oui, vers `127.0.0.1` | non, l'IP est obligatoire |
| Certificat fourni | ~~oui~~ révoqué | non |
| Activité du projet | faible | active |

Le premier point m'importe peu : mes services tournent sur `10.0.0.10`, pas sur `127.0.0.1`, et l'IP figurait déjà dans mes noms. Si vous ne travaillez qu'en local sur votre propre machine, `myapp.traefik.me` reste plus court, et la partie DNS de traefik.me fonctionne toujours.

Le certificat, en revanche, aucun service DNS ne le remplace. C'est là que j'en suis resté : les noms ont changé, les services sont passés en HTTP, et le port 443 est resté publié dans le `compose.yaml` de Traefik, avec une ligne `443:443/udp` et son commentaire « HTTP/3 nécessite UDP ».

## Remettre le HTTPS avec mkcert

La solution recommandée par pyrou, c'est [mkcert][mkcert]. Au lieu d'un certificat public partagé, on crée sa propre autorité de certification, on la déclare de confiance sur ses machines, et on émet ses certificats avec. Personne d'autre n'a la clé, et personne ne peut la faire révoquer.

```bash
mkcert -install                    # crée l'autorité locale et l'ajoute aux magasins de certificats
mkcert "*.127-0-0-1.sslip.io"      # émet un certificat wildcard signé par cette autorité
```

Pour cet article, j'ai monté l'ensemble de bout en bout dans des conteneurs jetables : mkcert, Traefik et un service `whoami`, ouvert dans un vrai Firefox. C'est de là que viennent l'image en tête de l'article et celle-ci :

{{< figure src="images/traefik-local-https/verifie-par-mkcert.png" alt="Onglet Sécurité de Firefox pour whoami.127-0-0-1.sslip.io : vérifiée par mkcert development CA, connexion chiffrée en TLS 1.3" caption="Le navigateur accepte le certificat parce qu'il connaît l'autorité locale : « Vérifiée par : mkcert development CA », connexion en TLS 1.3." >}}

Côté Traefik, trois choses changent par rapport à une installation en HTTP. D'abord, un fichier de configuration dynamique qui déclare le certificat, chargé par le provider `file` :

```yaml
# config/tls.yml
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

Ensuite, un point d'entrée sur le port 443 dans la configuration statique (`--entrypoints.websecure.address=:443`), avec les dossiers `config/` et `certs/` montés dans le conteneur. Enfin, deux labels sur chaque service, pour qu'il réponde sur ce point d'entrée en TLS :

```yaml
traefik.http.routers.whoami.entrypoints: websecure
traefik.http.routers.whoami.tls: true
```

Le reste, le réseau partagé et les labels de routage, ne change pas. Il est décrit dans [l'article sur la conception du homelab]({{< relref "2026-05-17-homelab-docker-compose-traefik.md" >}}). Et le certificat ne dépend pas du DNS choisi : seul le nom passé à mkcert et la règle `Host` changent.

{{< tabs >}}
{{< tab label="sslip.io" >}}
```bash
mkcert "*.127-0-0-1.sslip.io"
```

```yaml
traefik.http.routers.whoami.rule: Host(`whoami.127-0-0-1.sslip.io`)
```
{{< /tab >}}
{{< tab label="traefik.me" >}}
```bash
mkcert "*.traefik.me"
```

```yaml
traefik.http.routers.whoami.rule: Host(`whoami.traefik.me`)
```
{{< /tab >}}
{{< /tabs >}}

## Ce qui reste à faire chez moi

mkcert a une limite, et elle pèse plus lourd dans un homelab que sur un poste de développeur : l'autorité n'est connue que des machines sur lesquelles on l'installe. `mkcert -install` s'occupe de la machine où on le lance. Pour toutes les autres, ordinateurs, téléphones, tablettes, il faut copier le fichier `rootCA.pem` (son emplacement est donné par `mkcert -CAROOT`) et l'importer à la main dans chaque appareil et parfois dans chaque navigateur. Un appareil oublié affiche un avertissement de sécurité à chaque visite.

Pour mon homelab, la liste est donc courte, mais elle touche toute la maison :

1. générer un certificat pour `*.10.0.0.10.sslip.io` ;
2. ajouter le point d'entrée `websecure` et le fichier `tls.yml` à Traefik, qui publie déjà le port 443 ;
3. passer les labels des services sur `websecure` ;
4. installer l'autorité sur chaque appareil qui consulte le homelab.

Les trois premières étapes tiennent dans un dossier, la quatrième dans chaque appareil de la maison.

{{< alert >}}
La clé de l'autorité mkcert (`rootCA-key.pem`) permet d'émettre un certificat valide pour n'importe quel site sur toutes les machines qui lui font confiance. Elle ne doit jamais quitter la machine où elle a été créée, ni être versionnée. C'est l'erreur de traefik.me, à l'échelle d'une maison.
{{< /alert >}}

Avec le recul, traefik.me était confortable parce qu'il supprimait une frontière de sécurité sans le dire : le certificat était valide partout parce que sa clé était à tout le monde. mkcert demande plus de travail, mais la confiance repose sur une clé que je suis le seul à détenir. sslip.io règle la partie DNS, mkcert la partie TLS, et aucun des deux ne dépend de la bonne volonté d'un tiers pour continuer à fonctionner.

[traefik-me]: https://github.com/pyrou/traefik.me
[sslip]: https://sslip.io
[mkcert]: https://github.com/FiloSottile/mkcert
