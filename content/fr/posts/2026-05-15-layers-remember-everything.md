---
title: "Les couches se souviennent de tout"
date: 2026-05-15T12:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 3
categories: [développement]
tags: [symfony, cloud, docker, kubernetes, 12factor, securite]
description: "Comment les credentials de dev se retrouvent dans les images Docker de production, et pourquoi le Factor III et V sont plus difficiles à appliquer qu'ils n'y paraissent."
---

Le fichier `.dockerignore` avait la bonne intuition :

```
.env.local
.env.*.local
.env.local.php
```

Quelqu'un y avait pensé. Il savait que `.env.local.php` — le cache d'environnement compilé que Symfony génère — ne devrait pas voyager d'une machine de développeur vers une image Docker. Donc il l'a exclu du build context.

Ce qu'il n'a pas anticipé, c'est que le build en génère un nouveau.

## La chaîne

Le Dockerfile de production suit un pattern multi-stage qui est, en apparence, bien structuré. Un stage de base installe les extensions PHP et les dépendances système. Un stage prod compile l'application sans dépendances dev, optimise l'autoloader, et exécute `composer dump-env prod` avant de passer la main à FrankenPHP :

```dockerfile
RUN set -eux; \
    composer install --no-cache --prefer-dist --no-dev --no-autoloader --no-scripts; \
    composer dump-autoload --classmap-authoritative --no-dev; \
    composer dump-env prod; \
    sync;
```

`composer dump-env prod` est une optimisation de performance légitime. Au lieu de parser les fichiers `.env` à chaque requête, Symfony charge un fichier PHP pré-compilé — `.env.local.php` — qui retourne directement un tableau de paires clé-valeur. Rapide, propre, idiomatique.

Mais il doit lire quelque chose pour compiler. Et ce qu'il lit, c'est le fichier `.env` qui a été copié dans l'image quelques étapes plus tôt via `COPY --link . ./`.

Ce fichier `.env` est commité dans le dépôt. Dans le service `media`, il contient :

```dotenv
APP__JWT_SECRET="w4kg7YL9Fk5zmzeapdYjQHyICT8G8JGt1JCoUS7j4Wf2JgczPu76wjKVrIZHRw7a"
APP__HTTP_CACHE__CDN_ACCESS_TOKEN="akab-65n77xfcazkbwtgy-zfakzr6xqak5ths6"
APP__HTTP_CACHE__CDN_CLIENT_SECRET="s+LMM+7QtotVhIdtHhyLjWC6fsYK8aQDUA8mb..."
APP__HTTP_CACHE__CDN_CLIENT_TOKEN="akab-nz3ctehn6l5bpz6f-ar2klnztmief3fq7"
```

Ces valeurs sont décrites comme des credentials de développement. Mais le format de token Akamai n'est pas quelque chose qu'on génère localement. Et un secret JWT commité dans un dépôt cesse d'être un secret dès que le premier développeur clone le projet.

`dump-env prod` les lit, les compile dans `.env.local.php`, et l'image embarque une copie mise en cache de chaque credential qui se trouvait dans `.env` au moment du build. Le `.dockerignore` qui excluait le `.env.local.php` du développeur ne touche pas celui que le build vient de créer.

## Ce que le Factor V veut vraiment dire

Le <a href="https://12factor.net/build-release-run" target="_blank" rel="noopener noreferrer">cinquième facteur</a> trace une ligne dure entre trois étapes :

- **Build** : transformer le code en bundle exécutable. La sortie est agnostique à l'environnement.
- **Release** : combiner le build avec la configuration spécifique au déploiement.
- **Run** : exécuter la release.

Le mot-clé est agnostique à l'environnement. Un artefact de build devrait être le même binaire qu'il parte vers le staging ou la production. On le tague, on le stocke, on le promeut. La config arrive à l'étape Release, injectée de l'extérieur — variables d'environnement, gestionnaires de secrets, Kubernetes Secrets.

Le Dockerfile multi-stage a Build et Run. Il saute Release comme concept. La config n'entre pas de l'extérieur ; elle est compilée à l'intérieur. Si le CI injecte les secrets de production au moment du build — pour éviter une étape d'injection séparée plus tard — ces valeurs sont gelées dans les couches de l'image, permanentes et extractibles. L'image n'est plus agnostique à l'environnement. C'est un artefact de staging ou de production, pas les deux.

## Ce que le Factor III veut vraiment dire

Le <a href="https://12factor.net/config" target="_blank" rel="noopener noreferrer">troisième facteur</a> est précis : la config est tout ce qui varie entre les déploiements. Elle devrait vivre dans des variables d'environnement — de vraies variables OS, pas des fichiers qui utilisent le mot "environnement" dans leur nom.

La plateforme avait déjà compris ça au niveau applicatif. Chaque clé de config suit une convention stricte — `APP__<CATÉGORIE>__<CLÉ>` — et la config Symfony lit depuis ces variables avec `%env(APP__...)%`. La structure est bonne.

Le manque est dans ce que "variable d'environnement" signifie au niveau infrastructure. Écrire `DATABASE_PASSWORD=dave` dans `.env` et le commiter dans git produit un fichier dont le contenu peut être lu par quiconque a accès au dépôt, quiconque exécute `docker inspect`, ou quiconque fait :

```bash
docker run --rm <image> php -r "var_dump(require '.env.local.php');"
```

Et même si on essayait de faire le ménage après coup, les couches Docker sont des archives immuables. Même si une étape `RUN rm .env` ultérieure supprime le fichier du filesystem du conteneur en cours d'exécution, la couche qui le contient existe toujours dans l'image. `docker save <image>` produit une archive tar de chaque couche ; de là, extraire n'importe quel fichier de n'importe quel point de l'historique de build est une question de décompresser la bonne couche. Le fichier est invisible à l'exécution. Il n'est pas supprimé.

Une vraie variable d'environnement, au sens twelve-factor, est injectée par le runtime — un Kubernetes Secret monté comme variable d'env, un sidecar de gestionnaire de secrets, un système CI qui écrit des valeurs dans l'environnement du conteneur sans qu'elles touchent jamais un fichier. L'application ne sait jamais d'où vient la valeur. L'image ne la contient jamais.

## La correction en pratique

Le changement est plus petit qu'il n'y paraît.

`.env` devrait contenir seulement des valeurs par défaut structurelles : strings vides, valeurs placeholder, valeurs de repli sûres pour le développement. La convention que la plateforme utilise déjà fonctionne parfaitement pour ça :

```dotenv
APP__JWT_SECRET=""
APP__HTTP_CACHE__CDN_ACCESS_TOKEN=""
APP__HTTP_CACHE__CDN_CLIENT_SECRET=""
```

`dump-env prod` compile ces valeurs vides dans `.env.local.php`. À l'exécution, Kubernetes injecte les vraies valeurs comme variables d'environnement OS, qui ont la priorité. L'image est agnostique à l'environnement. Les credentials ne touchent jamais une couche.

Si on a genuinement besoin d'un secret pendant le build lui-même — un token pour s'authentifier contre un dépôt Composer privé, une clé SSH pour une dépendance git privée — BuildKit fournit une solution propre qui ne laisse aucune trace dans les couches :

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=composer_token \
    COMPOSER_AUTH=$(cat /run/secrets/composer_token) composer install
```

```bash
docker build --secret id=composer_token,src=./token.txt .
```

Le secret est disponible pendant l'étape `RUN` et disparaît quand elle se termine. Il n'apparaît jamais dans `docker history`, jamais dans aucune couche.

Le `.dockerignore` montre déjà que l'équipe pensait dans la bonne direction. L'instinct d'exclure `.env.local.php` était correct — il avait juste besoin d'être appliqué une étape en amont : garder les vrais credentials hors de `.env`, et la sortie compilée s'occupe d'elle-même.

## Construire une fois, déployer partout

L'objectif de séparer Build de Release est qu'une seule image peut être promue à travers les environnements sans reconstruction. On construit une fois, on teste cet artefact, on le déploie en staging, on promeut la même image en production. L'image est l'unité de confiance.

Cette propriété se casse dès que des valeurs spécifiques à l'environnement sont compilées à l'intérieur. Une image buildée avec des secrets de staging n'est pas le même artefact qu'une buildée avec des secrets de production, même si le code est identique. On ne promeut plus l'artefact testé ; on en construit un nouveau en espérant qu'il se comporte de la même façon.

Les deux premiers articles de cette série — [le cache filesystem](/2026/05/15/the-cache-that-was-lying-to-us/) et [le buffer de logs](/2026/05/15/no-witnesses/) — décrivaient des processus qui fonctionnaient bien sur une instance et s'effondraient quand on en multipliait. C'est le même pattern au niveau de l'image : ça fonctionne bien quand on déploie vers un seul environnement, et les coutures apparaissent quand on essaie de le promouvoir à travers eux.

L'image devrait être ignorante des credentials. L'infrastructure devrait être intelligente pour les injecter. C'est la division de responsabilité que le Factor III et le Factor V pointent — et `dump-env prod`, utilisé correctement, est parfaitement compatible avec les deux.
