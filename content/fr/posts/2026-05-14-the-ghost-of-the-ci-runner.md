---
title: "Le fantôme du runner CI"
date: 2026-05-14T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 1
categories: [développement]
tags: [symfony, cloud, storage, flysystem, minio, kubernetes, 12factor, ci]
description: "Comment un chemin pointant vers un runner CI dans la config de production a révélé une dette de stockage — et comment les lazy adapters Flysystem l'ont résolue."
---

```dotenv
APP__COLD_STORAGE__FILESYSTEM_PATH="/home/jenkins-slave/share_media/media"
APP__COLD_STORAGE__FILESYSTEM_PATH_CACHE="/home/jenkins-slave/share_media/media/cache"
APP__COLD_STORAGE__RAW_IMAGE_PATH="/home/jenkins-slave/share_media/media_raw"
APP__SHARE_STORAGE__FILESYSTEM_PATH="/home/jenkins-slave/share_storage"
```

Ces lignes se trouvaient dans le `.env` de production du service media. Pas le staging. Pas un override local. La production, committée dans le dépôt, lue à chaque démarrage.

Les chemins se terminent là où on s'y attendrait : `/media`, `/share_storage`. Ils commencent ailleurs : `/home/jenkins-slave`, le répertoire home d'un runner CI issu d'une ancienne installation Jenkins.

## Comment le home d'un runner atterrit dans la config de production

La plateforme avait grandi depuis une seule machine. Un serveur faisait tout tourner — l'application, le runner CI, la base de données, le stockage de fichiers. Les fichiers transitaient entre l'app et le système CI via NFS : un répertoire monté sur le même hôte, accessible aux containers comme au runner.

Le chemin `/home/jenkins-slave/share_media` était là où le partage NFS atterrissait sur cette machine. Quand l'équipe a migré vers Docker Compose, les containers ont hérité du montage NFS. Le chemin est entré dans le `.env` parce que l'application devait savoir où trouver les fichiers. Personne ne l'a changé parce que ça marchait. Le montage était toujours là. Le chemin était valide. L'application démarrait. Les fichiers apparaissaient où ils devaient.

Trois ans plus tard, personne n'y pensait plus du tout. C'était juste comme ça que le chemin media était configuré.

## Ce que kubectl apply a trouvé

Le premier `kubectl apply` du service media s'est terminé avec un pod bloqué en CrashLoopBackOff. Le container démarrait. L'entrypoint tournait. L'application essayait d'accéder à `/home/jenkins-slave/share_media/media`. Fichier ou répertoire inexistant. Pas de montage NFS. Pas de runner.

Le chemin ne documentait pas une décision de design. Il documentait la machine qui tournait par hasard au moment où le `.env` avait été écrit.

C'est exactement le problème que le [Facteur IV](https://12factor.net/backing-services) de l'application twelve-factor décrit. Les backing services — stockage, files, bases de données — doivent être des ressources attachées, configurées via URL ou chaîne de connexion, interchangeables entre environnements sans toucher au code. Un chemin de fichier sur un hôte partagé n'est pas un backing service. C'est une hypothèse physique sur la machine. Quand la machine change, l'hypothèse lâche.

## Le chemin était le symptôme

La première étape évidente était de supprimer la référence au runner :

```dotenv
APP__COLD_STORAGE__FILESYSTEM_PATH="/share_media/media"
APP__SHARE_STORAGE__FILESYSTEM_PATH="/share_storage"
```

Plus propre. Plus de références CI dans une config de production. Toujours incorrect. L'application supposait encore un système de fichiers POSIX — soit un volume monté, soit un répertoire sur le nœud. Dans Kubernetes, un volume partagé entre plusieurs pods nécessite un PersistentVolumeClaim en mode `ReadWriteMany`. La plupart des fournisseurs de stockage ne le supportent pas. Ceux qui le font ont tendance à être lents et coûteux. Et même là où ça fonctionne, on a juste remplacé une hypothèse sur le système de fichiers par une autre.

Renommer le chemin gagnait du temps. Ça ne réglait pas le problème.

Le problème, c'est qu'environ douze téraoctets d'images — originaux et déclinaisons pré-générées dans différents formats — pour plusieurs marques éditoriales — étaient traités comme un répertoire. Un répertoire ne se monte pas proprement sur plusieurs pods. Un backing service, si.

## Flysystem comme forme de la solution

Le service media avait déjà Flysystem de configuré. Trois adaptateurs concrets — système de fichiers local, AWS S3, Azure Blob — et un adaptateur lazy par-dessus :

```yaml
# config/packages/flysystem.yaml
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

        media.storage:
            adapter: 'lazy'
            options:
                source: '%env(APP__FLYSYSTEM_MEDIA_STORAGE)%'
```

Tout le code de l'application dépend de `media.storage`. Il ne sait pas si les fichiers vivent sur le système de fichiers ou dans un bucket cloud. Une variable d'environnement détermine quel backend est actif :

```dotenv
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.aws   # production
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.local  # fallback local toujours disponible
```

Le chemin est parti. L'hypothèse sur le système de fichiers est partie. Ce qui reste, c'est un nom de service — une ressource attachée au sens twelve-factor, configurable sans rebuilder l'image.

Le même pattern s'étend au cache de vignettes. [LiipImagine](https://github.com/liip/LiipImagineBundle) génère des images redimensionnées à la demande ; les originaux et le cache généré passent par des adaptateurs Flysystem séparés :

```yaml
liip_imagine:
    loaders:
        default:
            flysystem:
                filesystem_service: 'media.storage'
        default_cache:
            flysystem:
                filesystem_service: 'media.cache.storage'
```

Deux variables d'environnement, deux buckets. Toute la chaîne — recevoir l'upload, stocker l'original, générer la vignette, la mettre en cache — est portable vers le cloud sans toucher une ligne de PHP.

Ce que l'article ne couvre pas, c'est le déplacement des données. Le lazy adapter change une variable d'environnement. Faire passer douze téraoctets d'un montage NFS vers un bucket S3, c'est un autre projet — une fenêtre de migration, une double-écriture pendant le cutover, une vérification qu'il ne manque rien.

## Ce que Minio rend possible en CI

La production utilise S3. Le développement local utilise [Minio](https://min.io/), un stockage objet compatible S3 qui tourne dans un container Docker. L'adaptateur AWS parle à Minio en local et à S3 en production. L'application ne voit pas la différence :

```dotenv
# local/CI
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.aws
APP__MINIO_ENDPOINT=http://minio:9000
APP__MINIO_ACCESS_KEY=minioadmin
APP__MINIO_SECRET_KEY=minioadmin
```

Le même code, le même adaptateur, un endpoint différent. Pas de mock, pas de chemins de test spéciaux, pas de branches conditionnelles par environnement.

Mais la configuration CI va un cran plus loin. L'image Minio utilisée dans le pipeline n'est pas l'image officielle upstream — c'est une image custom buildée avec des fixtures de test préchargées :

```dockerfile
FROM minio/minio:latest
COPY tests/fixtures/ /fixtures_media/
```

Chaque run CI démarre avec une instance Minio qui contient déjà les données attendues par la suite de tests. Pas de script de setup, pas de commande de seed, pas d'étape "attendre le chargement des fixtures" avant que les tests commencent. L'état initial de l'environnement de test fait partie de l'artefact de build.

Le [Facteur V](https://12factor.net/build-release-run) appliqué à l'infrastructure de test : l'état de l'environnement est buildé, versionné, immuable. Le pipeline CI construit l'image Minio depuis la même source et au même commit que l'image applicative. Les fixtures de test et le code qui les exploite sont toujours synchronisés.

## Le compromis S3, honnêtement

S3 introduit un coût de latence que le stockage local n'a pas. Les premières données d'un fichier prennent 10 à 30 millisecondes à arriver depuis S3 — c'est la latence first-byte documentée du service, pas une mesure sur ce trafic spécifique.

À 300 requêtes par seconde, le raisonnement pour accepter ce compromis était le suivant : la majorité des lectures touche des vignettes déjà générées dans le cache S3, pas les fichiers originaux. Une image fraîchement uploadée paie la pénalité du cold miss une fois, à la première demande de vignette. Tout ce qui suit est un cache hit. Savoir si la latence de queue sous charge réelle confirmait ce raisonnement nécessitait des tests de charge suivis séparément — la décision d'architecture et la validation étaient découplées.

Le compromis a été accepté : comportement prévisible sur plusieurs pods, pas de problèmes d'état partagé, une couche de stockage qui scale sans coordination. L'histoire complète des mesures appartient au rapport de tests de performance, pas ici.

## Le fantôme s'en va

Le chemin `/home/jenkins-slave` n'apparaît plus dans la configuration. Mais ce à quoi il pointait était un couplage qui précédait Docker, précédait les microservices, précédait n'importe quelle conversation sur la migration cloud. Le runner CI et l'application de production partageaient un système de fichiers parce qu'ils vivaient sur la même machine. Personne ne l'avait conçu comme ça. Ça s'était accumulé.

Une erreur `kubectl apply` sur un chemin qui n'aurait pas dû exister a forcé la question : pourquoi cette application suppose-t-elle qu'un runner CI spécifique est présent sur l'hôte ? La réponse était "parce que ça a toujours été comme ça." Ce n'est pas une raison. C'est une histoire.

Renommer le chemin était un correctif en carton. L'adaptateur lazy de Flysystem était la vraie réponse — pas parce qu'il est plus élégant, mais parce qu'il fait du backend de stockage une décision qui appartient à l'environnement, pas à l'application. Le container démarre, lit une variable, se connecte à ce qui est à l'autre bout. Il ne sait pas si c'est un bucket dans un datacenter ou un container sur un laptop.

Le répertoire home du runner a disparu de la config. Ce qui l'a remplacé, c'est un nom de service. C'est la différence.
