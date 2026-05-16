---
title: "Trois Adaptateurs, Une Variable"
date: 2026-05-15T14:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 5
categories: [développement]
tags: [symfony, cloud, flysystem, s3, kubernetes, 12factor]
description: "Comment l'adaptateur lazy de Flysystem a transformé le déplacement du stockage media d'un volume Docker partagé vers S3 en un changement d'une ligne de config — et pourquoi l'abstraction était déjà là."
---

Le fichier `compose.yaml` à la racine du projet déclarait deux volumes nommés :

```yaml
volumes:
    share_storage:
    share_media:
```

Ces volumes étaient montés simultanément sur plusieurs services. Le service `bam` écrivait des fichiers dans `share_storage`. Le service `media` lisait depuis `share_media` et y réécrivait des vignettes. Le service `conversion` avait besoin des deux. Dans Docker Compose, c'est invisible — un volume nommé est juste un répertoire sur le host, partagé de façon transparente entre les conteneurs.

Dans Kubernetes, un volume partagé entre plusieurs pods nécessite un PersistentVolumeClaim `ReadWriteMany`. Tous les fournisseurs de stockage ne le supportent pas. Quand ils le font, ça tend à être cher et lent. Quand ils ne le font pas, on le découvre au pire moment possible.

Mais avant que tout ça devienne un problème, quelqu'un avait déjà construit la trappe de sortie.

## Le pattern déjà dans le code

Le service `media` a une configuration Flysystem qui ressemble à un catalogue :

```yaml
flysystem:
    storages:
        media.storage.local:
            adapter: 'local'
            options:
                directory: "/"

        media.storage.aws:
            adapter: 'aws'
            options:
                client: 'aws_client_service'
                bucket: 'media'
                streamReads: true

        media.storage.azure:
            adapter: 'azure'
            options:
                client: 'azure_client_service'
                container: 'media'

        media.storage:
            adapter: 'lazy'
            options:
                source: '%env(APP__FLYSYSTEM_MEDIA_STORAGE)%'
```

Trois adaptateurs concrets — filesystem local, AWS S3, Azure Blob — et un adaptateur `lazy` par-dessus eux. L'adaptateur lazy ne fait rien lui-même ; il délègue à l'adaptateur que la variable d'environnement nomme.

Tout le code applicatif dépend de `media.storage`. Il ne sait pas, et n'a pas besoin de savoir, si ses fichiers vivent sur le filesystem, dans S3, ou dans Azure Blob Storage. Cette décision vit dans une seule variable d'environnement :

```dotenv
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.aws
```

C'est exactement ce que décrit le <a href="https://12factor.net/backing-services" target="_blank" rel="noopener noreferrer">Factor IV</a> : les services de support comme des ressources attachées, interchangeables via la configuration. Le même code, pointé vers un endpoint différent par une valeur différente. Pas de reconstruction, pas de changement de code.

## Ce que les volumes partagés cachaient

Le volume `share_media` fonctionnait parce que les fichiers écrits par un conteneur apparaissaient dans le filesystem d'un autre conteneur au même chemin. Deux services pouvaient partager du media sans se connaître. Ça ressemblait à un filesystem — parce que c'en était un.

Le problème est qu'un filesystem n'est pas un service de support au sens twelve-factor. C'est de l'état local. C'est la même classe de problème que l'adaptateur de cache du [premier article de cette série](/2026/05/15/the-cache-that-was-lying-to-us/) : invisible sur un nœud, cassé sur deux.

Scaler le service `media` à deux pods dans Kubernetes. Le Pod A reçoit un upload, le stocke sur le filesystem local. Le Pod B reçoit la requête pour servir ce fichier. Le Pod B a un filesystem vide. Le fichier n'existe pas du point de vue du Pod B.

Le pattern de volume partagé masque ce problème dans Docker Compose parce qu'il n'y a qu'un seul host, et les deux conteneurs montent le même répertoire sur ce host. Dès qu'on introduit un second nœud — ou un second pod sur un nœud différent — l'illusion se brise.

Avec Flysystem et `media.storage.aws`, les deux pods lisent et écrivent dans le même bucket S3. Le filesystem est parti. <a href="https://12factor.net/processes" target="_blank" rel="noopener noreferrer">Factor VI</a> satisfait — par configuration, pas par code.

## LiipImagine était déjà câblé

Le service `media` génère des vignettes d'images à la demande avec LiipImagine. Ça ajoute une couche : non seulement le media original doit vivre quelque part d'accessible, mais le cache généré des images redimensionnées aussi. Dans un setup multi-pods, une vignette générée par le Pod A doit être trouvable par le Pod B.

La configuration montre que quelqu'un y avait déjà réfléchi :

```yaml
liip_imagine:
    loaders:
        default:
            flysystem:
                filesystem_service: 'media.storage'
        default_cache:
            flysystem:
                filesystem_service: 'media.cache.storage'

    data_loader: 'default'
    cache: 'default_cache'
```

Le loader source et le resolver de cache passent tous deux par Flysystem. `media.storage` est l'adaptateur lazy pour les originaux. `media.cache.storage` est un adaptateur lazy séparé pour les vignettes générées, soutenu par son propre bucket :

```dotenv
APP__FLYSYSTEM_MEDIA_CACHE_STORAGE=media.cache.storage.aws
```

Le pipeline complet — recevoir l'upload, stocker l'original, générer la vignette à la première requête, mettre en cache la vignette pour les requêtes suivantes — est portable vers le cloud sans toucher une ligne de PHP. La transformation est déjà cloud-native ; elle avait juste besoin d'être pointée vers le bon bucket.

## L'environnement de dev comme preuve

Le setup de développement local utilise <a href="https://min.io/" target="_blank" rel="noopener noreferrer">Minio</a>, un stockage objet compatible S3 qui tourne dans Docker. Le même adaptateur AWS cible S3 en production et le conteneur Minio en développement — seul l'endpoint change.

Ce qui signifie que l'intégration entre l'application et S3 était testable en local avant même que la migration cloud commence. Pas de mock, pas d'adaptateurs de test spéciaux, pas de surprises "ça marche en prod mais pas en dev". La surface d'API de S3 était déjà le contrat — Minio l'émulait juste sur un laptop.

Quand les credentials cloud remplacent les credentials Minio locaux dans l'environnement, le code ne s'en aperçoit pas. Le bucket répond à la même API. L'application s'en fiche.

## Ce qui restait

Les volumes partagés étaient encore présents dans `compose.yaml` pour les services qui n'avaient pas complètement basculé. Quatre services montent encore l'un ou les deux :

| Service | Volumes | Adaptateurs Flysystem | Statut |
|---|---|---|---|
| `media` | `share_storage`, `share_media` | local / aws / azure | cloud-configuré, volumes vestigiaux |
| `bam` | `share_storage` | local / aws / azure | adaptateurs prêts, env var à basculer |
| `conversion` | `share_storage` | local / aws / azure | adaptateurs prêts, env var à basculer |
| `sitemap` | `share_media` | local / aws | adaptateurs prêts, env var à basculer |

Pour `media`, les adaptateurs Flysystem pointent déjà vers le stockage cloud — les mounts de volume restent comme fallback pour les chemins de stockage froide pas encore routés via l'abstraction. Pour les trois autres, le code est prêt ; la migration est une env var par service.

Contrairement au `.dockerignore` dans [l'article sur les secrets](/2026/05/15/layers-remember-everything/), où la bonne intuition est arrivée une étape trop tard, ici l'abstraction a été construite avant qu'il y ait une pression pour l'utiliser. L'abstraction a été construite avant que la migration soit planifiée. Ce n'est pas courant. En général, la décision d'aller sur le cloud arrive en premier, l'abstraction arrive sous pression, et le code montre les coutures. Ici les coutures sont propres parce que le modèle d'adaptateur de Flysystem a exactement la bonne forme pour ce problème : une interface, plusieurs backends, aucune logique métier qui se soucie lequel est actif.

Les volumes partagés dans Docker Compose n'étaient pas une erreur de conception. C'était un choix raisonnable pour un environnement où seul un host tournait. L'erreur aurait été de supposer qu'un volume partagé est une architecture permanente plutôt qu'une commodité locale. L'adaptateur lazy est ce qui fait la différence : quand le volume n'est plus la bonne réponse, l'application n'a pas besoin d'être informée. La variable d'environnement l'est.
