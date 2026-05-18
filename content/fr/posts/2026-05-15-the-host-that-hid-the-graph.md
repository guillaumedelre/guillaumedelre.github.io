---
title: "L'hôte qui cachait le graphe"
date: 2026-05-15T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 4
categories: [développement]
tags: [symfony, cloud, kubernetes, 12factor, microservices, http-client]
description: "Treize services partageant six variables de gateway identiques. La config semblait simple. Le graphe de dépendances était invisible — jusqu'à ce que Kubernetes demande où chaque service habitait vraiment."
---

Chaque service de la plateforme avait ces six variables :

```bash
APP__GATEWAY__PRIVATE__HOST="platform.internal"
APP__GATEWAY__PRIVATE__PORT=80
APP__GATEWAY__PRIVATE__SCHEME="http"
APP__GATEWAY__PUBLIC__HOST="platform.internal"
APP__GATEWAY__PUBLIC__PORT=80
APP__GATEWAY__PUBLIC__SCHEME="http"
```

Treize services, six variables chacun, une seule valeur. En lisant la config d'un service quelconque, l'architecture semblait plate. Tout parlait au même hôte. C'était tout le tableau.

Ce ne l'était pas.

## Comment fonctionnait la gateway

La gateway se trouvait devant chaque service et gérait tout le trafic inter-services. Un service appelant l'API content construisait une requête vers `http://platform.internal/content/api/` — la gateway la recevait, identifiait la cible depuis le chemin de l'URL, et la transmettait au bon backend. Chaque client HTTP inter-service dans `framework.yaml` suivait le même schéma :

```yaml
content.client:
    base_uri: "%http_client.gateway.base_uri%/content/api/"
    headers:
        Host: "%env(APP__GATEWAY__PRIVATE__HOST)%"
```

Le paramètre `http_client.gateway.base_uri` était assemblé depuis les variables GATEWAY. La gateway savait où tournait chaque service. Les services n'avaient pas besoin de le savoir. De leur point de vue, tout était `platform.internal`.

Ça fonctionnait. Pendant des années, ça fonctionnait bien. Ajouter un service signifiait ajouter un alias DNS dans la config de la gateway, pas toucher treize fichiers `.env`. La gateway abstraisait la topologie. Les services restaient découplés du détail d'infrastructure de qui tournait où.

## Ce que la gateway absorbait

L'abstraction avait un coût qui n'apparaissait pas tant qu'on n'essayait pas de lire le système.

En regardant le fichier env de `content`, on voyait six variables de gateway et rien d'autre sur la communication inter-services. Pour découvrir que `content` appelait `conversion`, `shorty` et `media`, il fallait lire `framework.yaml`. Pour découvrir que `pilot` appelait dix services externes, il fallait tracer les clients HTTP un par un et compter.

Le chiffre était dix. Authentication, bam, config, content, conversion, media, product, shorty, sitemap, social. Dix des treize services de la plateforme dont `pilot` dépendait à l'exécution, aucun d'eux visible depuis sa configuration. Six variables disaient : parle à la gateway. Elles ne disaient rien de la forme de ce qui se trouvait derrière.

Cette information existait — dans le code, dans la config framework, dans les têtes des gens qui avaient construit ces intégrations. Elle ne vivait juste nulle part où on pouvait la lire d'un coup d'œil.

## Ce que Kubernetes a rendu explicite

On-premise, la gateway était un seul hostname résolvable. Un enregistrement DNS, un jeu de variables, un seul endroit à mettre à jour. Kubernetes ne fonctionne pas comme ça. Chaque service obtient son propre nom DNS à l'intérieur du cluster — `content.namespace.svc.cluster.local`, `conversion.namespace.svc.cluster.local`. Le trafic inter-services passe directement, service à service, sans gateway partagée.

Passer à Kubernetes signifiait que l'abstraction de la gateway devait céder la place. Chaque service devait savoir, concrètement, où vivait chacune de ses dépendances. Les six variables génériques ne pouvaient pas exprimer ça.

Le refacto les a remplacées par des variables HOST par cible — une par dépendance de service, nommée d'après la cible :

```bash
# content/.env — content appelle ces quatre services
APP__CONFIG__HOST="platform.internal"
APP__CONVERSION__HOST="platform.internal"
APP__MEDIA__HOST="platform.internal"
APP__SHORTY__HOST="platform.internal"
```

```bash
# pilot/.env — dix dépendances de service
APP__AUTHENTICATION__HOST="platform.internal"
APP__BAM__HOST="platform.internal"
APP__CONFIG__HOST="platform.internal"
APP__CONTENT__HOST="platform.internal"
APP__CONVERSION__HOST="platform.internal"
APP__MEDIA__HOST="platform.internal"
APP__PRODUCT__HOST="platform.internal"
APP__SHORTY__HOST="platform.internal"
APP__SITEMAP__HOST="platform.internal"
APP__SOCIAL__HOST="platform.internal"
```

Chaque client HTTP dans `framework.yaml` a reçu sa propre `base_uri` construite depuis la variable HOST de sa cible, et le header `Host` a cédé la place à un `User-Agent` qui identifie l'appelant :

```yaml
content.client:
    base_uri: "%env(APP__HTTP__SCHEME)%://%env(APP__CONTENT__HOST)%:%env(APP__HTTP__PORT)%/content/api/"
    headers:
        User-Agent: "Platform Content - %semver%"
```

Le changement n'est pas cosmétique. Dans l'ancienne configuration, le header `Host` explicite garantissait que les requêtes atteignaient le bon virtual host de la gateway quelle que soit la résolution DNS. Dans la nouvelle, chaque client pointe directement vers le nom DNS de sa cible — le `Host` correct est dérivé automatiquement de la `base_uri`. L'emplacement du header ne reste pas vide : le `User-Agent` identifie désormais le service appelant, ce qui remonte dans les logs et le traçage distribué sans instrumentation supplémentaire.

## L'inconfort de la lisibilité

Le fichier env de `pilot` est passé de neuf variables de gateway à dix variables HOST spécifiques par service. Le fichier est devenu plus long. L'architecture n'est pas devenue plus simple — les dix dépendances étaient là avant et elles sont toujours là. Ce qui a changé, c'est qu'elles sont lisibles.

Le [Facteur III](https://12factor.net/config) dit de stocker la config dans l'environnement. L'ancienne approche satisfaisait ça à la lettre : six variables, toutes dans des fichiers env, aucune en dur dans le code. Mais des variables qui effondrent le graphe de dépendances entier dans un seul hostname opaque ne sont pas vraiment de la configuration — elles sont un raccourci qui échange la lisibilité contre la commodité. Le Facteur III ne demande pas seulement que la config soit externalisée — il suppose implicitement qu'elle reste informative une fois externalisée.

Le refacto n'a rien simplifié. Il a rendu la complexité visible. Les dix variables HOST de `pilot` documentent, dans le fichier `.env` lui-même, les dix services dont il dépend. Un nouveau membre d'équipe qui lit ce fichier apprend quelque chose de réel sur l'architecture. L'ancien fichier lui apprenait qu'il y avait une gateway.

Il y a une version de cette histoire où on lit l'état final et on conclut que l'équipe a fait un travail inutile — elle a remplacé six variables par dix, toutes pointant vers le même hôte de toute façon. En développement local, `platform.internal` résout toujours au même endroit. Le comportement fonctionnel n'a pas changé.

Le changement est dans ce que la config communique. Dans Kubernetes, les valeurs HOST divergent : chaque cible obtient son propre nom DNS interne au cluster, différent par environnement. Les variables portent maintenant une vraie information. Le refacto a préparé la config à être honnête sur une topologie qu'elle simplifiait silencieusement depuis des années.
