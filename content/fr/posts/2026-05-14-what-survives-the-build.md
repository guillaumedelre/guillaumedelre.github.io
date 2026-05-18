---
title: "Ce qui survit au build"
date: 2026-05-14T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 2
categories: [développement]
tags: [symfony, cloud, docker, kubernetes, 12factor, security, secrets]
description: "Comment des fichiers .env committés alimentent un build qui grave les credentials dans les layers Docker — et ce qu'il faut pour vider le fichier jusqu'à quatre lignes."
---

À un moment de l'audit de migration cloud, quelqu'un a lancé ça :

```bash
docker run --rm <image> php -r "var_dump(require '.env.local.php');"
```

La sortie montrait tout ce que `composer dump-env prod` avait compilé dans l'image au moment du build. Ce qui voulait dire tout ce qui se trouvait dans le fichier `.env` quand l'image avait été construite. Ce qui voulait dire, entre autres, ça :

```dotenv
INFLUXDB_INIT_ADMIN_TOKEN=<influxdb-admin-token>
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=admin123
BLACKFIRE_CLIENT_ID=<blackfire-client-id>
BLACKFIRE_CLIENT_TOKEN=<blackfire-client-token>
BLACKFIRE_SERVER_ID=<blackfire-server-id>
BLACKFIRE_SERVER_TOKEN=<blackfire-server-token>
NGROK_AUTHTOKEN=replace-me-optionnal
```

Vingt-cinq variables au total. Chaque credential accumulé dans le `.env` racine sur trois ans, désormais permanent dans un layer d'image.

## Comment `dump-env` fonctionne

`composer dump-env prod` est une optimisation Symfony légitime. Au lieu de parser les fichiers `.env` à chaque requête, le runtime charge un tableau PHP pré-compilé depuis `.env.local.php`. Plus rapide et plus simple.

Le problème, c'est ce qu'il lit. Le Dockerfile copie le dépôt dans l'image avec `COPY . ./`, `.env` inclus. Ensuite `dump-env prod` lit ce fichier et compile chaque variable dans `.env.local.php`. L'image est livrée avec une capture figée des credentials qui se trouvaient dans `.env` au moment du build.

Les layers Docker sont des archives immuables. Même si une étape ultérieure supprimait `.env` du système de fichiers du container, le layer qui le contient existerait toujours dans l'image. `docker save <image>` produit une archive tar de chaque layer ; extraire un fichier spécifique de n'importe quel point de l'historique de build est une opération simple. Les credentials sont invisibles à l'exécution. Ils ne sont pas partis.

Le [Facteur V](https://12factor.net/build-release-run) est explicite là-dessus : un artefact de build doit être agnostique à l'environnement, la config arrivant à l'étape de release depuis l'extérieur. Dès que des credentials sont compilés dedans, l'image n'est plus portable. On ne peut plus la promouvoir entre environnements. On builde deux fois en espérant que le deuxième se comporte comme le premier.

## Comment vingt-cinq variables s'accumulent

Avant de voir comment on a réparé ça, il vaut la peine de comprendre comment on en est arrivé là.

Les tokens `BLACKFIRE_*` sont le cas facile à comprendre. Un membre de l'équipe configure le profiling, a besoin de partager la configuration, et le dépôt est déjà ouvert à tout le monde. Une ligne dans `.env` est la voie de moindre résistance. Les credentials InfluxDB et Grafana suivent la même logique — outillage partagé, dépôt partagé, un commit.

Puis il y a les variables qui révèlent une autre dérive. Dans certains `.env` de services :

```dotenv
APP__RATINGS__SERIALS='{"marque1":{"fr":"12345"},...}'  # ~40 lignes de JSON
APP__YOUTUBE__CREDENTIALS='{"marque1":{"client_id":"xxx","refresh_token":"yyy"},...}'
```

Des numéros de série pour la mesure d'audience. Des refresh tokens YouTube par marque. Ce ne sont pas des secrets au sens des tokens Blackfire. Ce sont des données métier — le genre de valeurs qui varient entre marques et environnements, que quelqu'un a décidé de versionner dans `.env` parce qu'elles se comportaient comme de la configuration et que `.env` était l'endroit où vivait la configuration.

Vingt-cinq variables, c'est la somme de décisions incrémentales, dont aucune ne semblait fausse isolément. Le problème est structurel : quand `.env` est la seule réponse disponible, tout finit par y ressembler.

## Où les choses appartiennent vraiment

Vider le fichier exigeait de répondre à une question pour chaque variable : *où est-ce que ça appartient vraiment ?*

Les réponses ont révélé trois catégories que l'équipe n'avait jamais explicitement nommées :

**La config statique** vit dans le code. Règles métier, logique de routing, fichiers de paramètres Symfony — tout ce qui ne varie pas entre les déploiements. Un changement exige un rebuild. Les blocs JSON de numéros de série se sont révélés ne pas être de la config statique du tout : ils étaient interrogés depuis un service Config dédié à l'exécution. Ils n'avaient rien à faire dans un fichier.

**La config environnementale** varie entre les déploiements : hostnames, chaînes de connexion, credentials de services tiers. C'est ce que le [Facteur III](https://12factor.net/config) désigne par "config dans les variables d'environnement" — de vraies variables au niveau OS, injectées à l'exécution, jamais des fichiers qui voyagent avec le code. Dans Kubernetes, c'est un ConfigMap pour les valeurs non sensibles et un Kubernetes Secret pour les credentials. Le choix retenu pour les secrets a été SOPS — les credentials sont chiffrés et committés dans git, plutôt que stockés dans un coffre-fort externe comme Azure Key Vault ou HashiCorp Vault. Un coffre-fort échange la simplicité contre l'auditabilité : rotation automatique, logs d'audit centralisés, accès via workload identity sans clé à protéger. SOPS échange ces capacités contre un modèle opérationnel plus simple — pas de service externe à interroger au déploiement, les secrets transitent par le processus de review normal du code, l'historique git fait office de piste d'audit. Les contreparties acceptées sont la rotation manuelle et la responsabilité de protéger la clé de déchiffrement elle-même. Pour la taille de l'équipe, le compromis était délibéré.

**La config dynamique** change sans déploiement : paramètres éditoriaux, seuils par marque, configuration de modération de contenu. Elle appartient à une base de données, gérée via le service Config de l'application. Une partie de ce qui s'était accumulé dans les `.env` de services était cette catégorie depuis le début, passant pour des valeurs par défaut statiques parce qu'elle changeait assez rarement pour que personne ne le remarque.

Une fois les catégories nommées, les variables se sont triées. Le `.env` racine est arrivé à quatre lignes :

```dotenv
DOMAIN=platform.127.0.0.1.sslip.io
XDEBUG_MODE=off
SERVER_NAME=:80
APP_ENV=dev
```

Des valeurs par défaut sûres. Rien de sensible. `dump-env prod` compile maintenant des chaînes vides ; les vraies valeurs arrivent à l'exécution depuis Kubernetes.

## L'image PostgreSQL

L'image PostgreSQL utilisée en CI a un mot de passe codé en dur :

```dockerfile
FROM postgres:15
ENV POSTGRES_PASSWORD=admin123
```

Ça ressemble au même problème. Ce n'en est pas un, parce que le modèle de menace est différent. La base CI est éphémère — elle existe le temps d'un run de pipeline, ne contient pas de vraies données, tourne dans un réseau isolé. Un mot de passe codé en dur sur une base de test jetable est un risque acceptable, pas une entorse à la règle.

En production, la question ne se pose pas : la plateforme utilise Azure Flexible Server, un service PostgreSQL managé. Il n'y a pas d'image Docker. Les credentials arrivent via injection dans les charts Helm, sans jamais toucher un layer.

## Ce qui survit au build maintenant

L'image qui part en production contient maintenant une garantie : `var_dump(require '.env.local.php')` ne retourne que des chaînes vides et des valeurs par défaut sûres. Les credentials ne sont pas là parce qu'ils n'y ont jamais été mis — ils arrivent à l'exécution, depuis l'extérieur.

C'est la frontière de responsabilité que `dump-env` avait silencieusement effacée : l'image est l'application, le runtime est l'environnement. Ils ne devraient pas connaître les secrets de l'autre.
