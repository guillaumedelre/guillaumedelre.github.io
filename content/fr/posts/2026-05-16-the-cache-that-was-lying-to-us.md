---
title: "Le cache qui nous mentait"
date: 2026-05-16T15:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 6
categories: [développement]
tags: [symfony, cloud, redis, cache, kubernetes, 12factor]
description: "Comment une seule ligne de config bloquait le scaling horizontal de 13 microservices Symfony, et ce que la méthodologie twelve-factor avait à dire là-dessus."
---

La première fois qu'on a lancé deux replicas du même service Symfony derrière un load balancer, tout avait l'air d'aller. Les health checks passaient. Le trafic se répartissait proprement. Les temps de réponse étaient bons.

Puis quelqu'un a remarqué que le rate limiter se comportait bizarrement. Cinq appels à l'API, accès bloqué. Cinq appels supplémentaires à la requête suivante, accès accordé. Selon quel pod répondait, on était une personne différente.

C'était le cache qui parlait. Une ligne de config, répliquée sur treize services, bloquait le scaling horizontal dans sa totalité.

## Un fichier de config, treize fois

On préparait une plateforme de treize microservices Symfony pour passer sur Kubernetes. La stack était déjà en bon état : FrankenPHP pour le serveur HTTP, des Dockerfiles multi-étapes, un GitLab CI qui poussait des images taguées vers un registre cloud. Les pièces étaient là. Il fallait juste vérifier que rien ne casserait quand on commencerait à scaler les pods horizontalement.

Une bonne checklist pour ce type d'audit, c'est la <a href="https://12factor.net" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor app</a> — douze principes pour construire des logiciels qui tournent proprement dans des environnements cloud. La plupart des facteurs étaient déjà couverts sans qu'on y ait pensé délibérément.

Le Facteur VII (port binding) était gratuit. FrankenPHP embarque Caddy directement dans le processus PHP. Le container expose son propre endpoint HTTP, sans Apache ni Nginx à ajouter. L'image est autonome, ce que le facteur demande exactement :

```dockerfile
HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
```

Le Facteur II (dépendances) était géré par `composer.json` et les extensions du Dockerfile. Le Facteur X (parité dev/prod) était suffisamment couvert pour notre périmètre : même image, mêmes backing services en local et en CI, ce qui est la partie qui compte vraiment pour l'audit.

Puis j'en suis arrivé au Facteur VI.

## Le problème avec « ça marche sur un seul serveur »

Le Facteur VI dit que les processus ne doivent rien partager. Rien d'écrit sur disque entre les requêtes, rien en mémoire locale qu'une autre instance ne puisse pas voir. Si on a besoin de persister de l'état, on le met dans un backing service — une base de données, un cluster de cache, une queue. Le processus lui-même reste jetable.

J'ai ouvert `authentication/config/packages/cache.yaml`. Puis `content/config/packages/cache.yaml`. Puis `media/config/packages/cache.yaml`.

```yaml
framework:
    cache:
        app: cache.adapter.filesystem
```

Treize services. Treize fois, mot pour mot.

Chaque instance de chaque service écrivait son cache sur le filesystem local. Ce qui signifiait que chaque pod avait son propre cache privé, invisible pour tous les autres pods. Quand le load balancer envoyait une requête au pod A, il obtenait la version mise en cache par le pod A. Le pod B avait construit la sienne. Elles pouvaient avoir été générées à des moments différents, depuis des données sources différentes, ou l'une d'elles pouvait ne pas encore avoir été construite du tout.

Le rate limiter était le symptôme le plus visible parce qu'il avait un compteur. Mais la même divergence affectait chaque donnée qu'on mettait en cache : métadonnées du sérialiseur, collections de routes, caches de résultats Doctrine. Deux utilisateurs envoyant des requêtes identiques pouvaient obtenir des réponses différentes selon quel nœud avait récupéré la connexion.

## Redis était déjà là

C'est la partie qui pique un peu. Redis était déjà dans la stack. Chaque service l'avait configuré via SncRedisBundle :

```yaml
# config/packages/snc_redis.yaml — présent sur les 13 services
snc_redis:
    clients:
        default:
            type: 'phpredis'
            alias: 'default'
            dsn: '%env(IN_MEM_STORE__URI)%'
```

Le Facteur IV de la twelve-factor app dit que les backing services doivent être des ressources attachées, interchangeables via la configuration. Redis était exactement ça : joignable via une variable d'environnement, prêt à être remplacé par une instance managée dans le cloud. La plomberie était faite. On ne s'en servait juste pas pour le cache applicatif.

Certains services l'avaient même juste pour des pools spécifiques. Le rate limiter dans le service d'authentification :

```yaml
pools:
    rate_limiter.cache:
        adapter: cache.adapter.redis
```

Ce qui explique l'incohérence qu'on a vue en premier. Le *compteur* du rate limit allait vers Redis (partagé entre les pods). Le cache qui alimentait la *vérification* du rate limit allait vers le filesystem (local au pod). Deux sources de vérité, l'une invisible à l'autre.

La correction tenait en une ligne par service :

```yaml
framework:
    cache:
        app: cache.adapter.redis
        default_redis_provider: snc_redis.default
```

Treize fichiers. Treize changements identiques. Le genre de correction qui donne l'impression qu'on aurait dû la repérer avant, sauf qu'elle est parfaitement invisible quand on tourne sur une seule instance.

## Ce qui doit migrer vers Redis

Le cache filesystem violait le Facteur VI (les processus portent de l'état local qu'ils ne devraient pas) et le Facteur VIII (on ne peut pas scaler sans partager cet état). C'est le même problème vu sous deux angles : VI décrit ce qui ne va pas, VIII décrit ce qu'on ne peut pas faire à cause de ça.

Avec un backend de cache partagé, un deuxième pod est sûr. Les deux pods construisent le même cache, voient les mêmes invalidations, s'accordent sur les mêmes limites de rate. On peut ajouter un troisième pod sous charge et le retirer quand le trafic baisse. L'orchestrateur s'en occupe ; l'application n'a pas besoin de le savoir.

Sans ça, le scaling horizontal est un risque. Plus de pods, c'est plus de divergence, plus de bugs « ça marche chez moi » qu'il est impossible de reproduire en local parce qu'en local on tourne avec un seul container.

Les sessions avaient le même problème — et potentiellement pire. Douze des treize services utilisaient `session.storage.factory.native` — qui écrit les sessions sur le filesystem par défaut. Un utilisateur dont la requête atterrit sur le pod A obtient une session liée au pod A. Sa requête suivante va sur le pod B. Session perdue, il est déconnecté. Un seul service avait `RedisSessionHandler` configuré.

La mitigation partielle : la plupart de la plateforme tourne sur des APIs stateless avec des JWT, donc l'usage des sessions est limité. Mais « limité » n'est pas « zéro ». Les services qui créent des sessions — flows d'authentification, état temporaire pendant les handshakes OAuth — ont un mode de défaillance visible par l'utilisateur qui attend le deuxième pod. Soit ces sessions migrent vers Redis, soit le code qui les crée est supprimé. Les laisser en l'état est une décision qui attend le premier utilisateur dont la session disparaît sans explication.

## L'autre genre d'état

Redis résout le problème cross-pod. FrankenPHP introduit un autre problème qu'il vaut la peine de connaître.

Dans le modèle PHP-FPM standard, chaque requête forke un processus frais. Tout objet en mémoire — toute valeur mise en cache, tout résultat calculé — meurt avec la réponse. Le processus est stateless par construction.

FrankenPHP a un mode worker qui ne suit pas ce modèle. En mode worker, un seul processus PHP démarre une fois, charge le kernel, câble le container, et gère plusieurs requêtes successives sans redémarrer. Le débit de requêtes s'améliore : pas de cold start de l'autoloader, pas de rebuild du container par requête, moins d'allocations. La contrepartie : le processus PHP a maintenant un cycle de vie qui enjambe les requêtes.

Pour le cache, ça ajoute une complexité. Un adaptateur `array` ou un pool APCu accumule des entrées à travers les requêtes sur le même worker. Une invalidation de cache poussée vers Redis atteint immédiatement les autres pods — mais ne vide pas ce qui est assis dans la mémoire du worker. Deux requêtes sur le même pod peuvent voir des choses différentes : l'une touche une entrée en mémoire chaude, la suivante déclenche un fetch Redis après expiration de l'entrée in-process.

La plateforme garde le mode worker désactivé (`APP__WORKER_MODE__ENABLED=false`). Il est disponible — l'infrastructure est là, le flag est câblé — mais pas actif. Le gain de performance ne justifiait pas l'audit. Chaque pool de cache aurait besoin d'être vérifié contre la sémantique du mode worker ; chaque endroit où de l'état fuit entre les requêtes deviendrait un bug potentiel.

La position conservatrice : garder PHP stateless au niveau du processus même quand le runtime ne l'exige pas. Le principe shared-nothing du Facteur VI s'applique non seulement au filesystem — il s'applique au processus lui-même.

## Ce qui fonctionnait déjà

Pour être juste envers la codebase : le Scheduler Symfony utilisait déjà Redis pour les locks distribués :

```php
$schedule->lock($this->lockFactory->createLock('schedule_purge'));
```

Dans un environnement multi-pod, on ne veut pas cinq instances lancer le même job de purge simultanément. Le lock l'empêche. Redis rend le lock visible entre les pods. Celui qui a écrit le scheduler savait exactement ce qu'il faisait.

Le même raisonnement ne s'était juste pas propagé à la configuration du cache — probablement parce qu'en tournant sur une seule instance, `cache.adapter.filesystem` est invisible. Ça fonctionne, c'est rapide, ça ne demande aucune configuration. Le problème n'apparaît qu'à deux.

## Les quatre questions

Le Facteur VI prend la plupart des applications par surprise lors d'une migration cloud. Pas parce que les développeurs ne connaissent pas les processus stateless — ils le savent généralement — mais parce que le filesystem est toujours là, et le problème reste caché jusqu'à ce qu'on essaie de lancer une deuxième instance.

Avant de scaler un service Symfony horizontalement, quatre questions méritent une réponse :

- Où va le cache applicatif ? (`cache.adapter.filesystem` doit devenir `cache.adapter.redis`)
- Où vont les sessions ? (`session.storage.factory.native` a besoin de Redis — ou supprimer les sessions entièrement si on est full JWT)
- Est-ce que quelque chose écrit dans `var/` à l'exécution qu'un autre pod aurait besoin de lire ?
- Est-ce qu'il y a quelque chose dans le chemin de code qui doit être mutuellement exclusif entre pods ? (si oui, c'est un job pour le <a href="https://symfony.com/doc/current/components/lock.html" target="_blank" rel="noopener noreferrer">composant Lock de Symfony</a> adossé à Redis, pas un mutex local)

Si toutes les réponses pointent vers des backing services partagés, on est prêt. Si l'une d'elles pointe vers le filesystem local, la production finira par trouver le pod qui a construit son cache il y a trois heures et le servira à l'utilisateur qui s'y attend le moins.
