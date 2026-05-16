---
title: "Le Cache Qui Nous Mentait"
date: 2026-05-15T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 1
categories: [développement]
tags: [symfony, cloud, redis, cache, kubernetes, 12factor]
description: "Comment une seule ligne de config a bloqué le scaling horizontal de 13 microservices Symfony, et ce que l'app twelve-factor avait à dire là-dessus."
---

La première fois qu'on a fait tourner deux répliques du même service Symfony derrière un load balancer, tout semblait nickel. Les health checks passaient. Le trafic se répartissait proprement. Les temps de réponse étaient bons.

Et puis quelqu'un a remarqué que le rate limiter se comportait bizarrement. Cinq appels à l'API, bloqué. Cinq appels de plus à la requête suivante, ça passe. Selon quel pod répondait, tu étais une personne différente.

C'était le cache qui parlait. Une seule ligne de config, répliquée sur treize services, bloquait entièrement le scaling horizontal.

## Un fichier de config, treize fois

On préparait une plateforme de treize microservices Symfony à migrer vers Kubernetes. La stack était déjà en bon état : FrankenPHP pour le serveur HTTP, des Dockerfiles multi-stage, un GitLab CI qui poussait des images taguées vers un registre cloud. Les pièces étaient là. Il fallait juste vérifier que rien ne casserait quand on commencerait à scaler les pods horizontalement.

Un bon point de départ pour ce genre d'audit, c'est la <a href="https://12factor.net" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor app</a> — douze principes pour construire des logiciels qui tournent proprement dans des environnements cloud. La plupart des facteurs étaient déjà couverts sans qu'on ait fait quoi que ce soit de délibéré à ce sujet.

Le facteur VII (port binding) était gratuit. FrankenPHP intègre Caddy directement dans le processus PHP. Le container expose son propre endpoint HTTP, pas besoin d'Apache ou Nginx à coller dessus. L'image est autonome, ce qui est exactement ce que le facteur demande :

```dockerfile
HEALTHCHECK --start-period=60s CMD curl -f http://localhost:2019/metrics || exit 1
```

Le facteur II (dépendances) était géré par `composer.json` et les extensions dans le Dockerfile. Le facteur X (parité dev/prod) était suffisamment couvert pour notre périmètre : même image, mêmes backing services en local et en CI, ce qui est la partie qui compte vraiment pour ce qu'on auditait.

Puis j'en suis arrivé au facteur VI.

## Le problème avec "ça marche sur un seul serveur"

Le facteur VI dit que les processus ne doivent rien partager. Rien écrit sur disque entre les requêtes, rien en mémoire locale qu'une autre instance ne peut pas voir. Si tu as besoin de persister un état, mets-le dans un backing service — une base de données, un cluster de cache, une queue. Le processus lui-même reste jetable.

J'ai ouvert `authentication/config/packages/cache.yaml`. Puis `content/config/packages/cache.yaml`. Puis `media/config/packages/cache.yaml`.

```yaml
framework:
    cache:
        app: cache.adapter.filesystem
```

Treize services. Treize fois, mot pour mot.

Chaque instance de chaque service écrivait son cache sur le filesystem local. Ce qui signifiait que chaque pod avait son propre cache privé, invisible aux autres pods. Quand le load balancer envoyait une requête au pod A, il obtenait la version cachée de la réalité du pod A. Le pod B en avait construit une autre. Elles pouvaient avoir été générées à des moments différents, à partir de données sources différentes, ou l'une d'elles n'avait peut-être pas encore été construite du tout.

Le rate limiter était le symptôme le plus visible parce qu'il avait un compteur. Mais la même divergence touchait toutes les données qu'on mettait en cache : les métadonnées du serializer, les collections de routes, les caches de résultats Doctrine. Deux utilisateurs envoyant des requêtes identiques pouvaient obtenir des réponses différentes selon quel nœud avait récupéré la connexion.

## Redis était déjà là

C'est là que ça pique un peu. Redis était déjà dans la stack. Chaque service l'avait configuré via SncRedisBundle :

```yaml
# config/packages/snc_redis.yaml — présent sur les 13 services
snc_redis:
    clients:
        default:
            type: 'phpredis'
            alias: 'default'
            dsn: '%env(IN_MEM_STORE__URI)%'
```

Le facteur IV de la twelve-factor app dit que les backing services doivent être des ressources attachées, interchangeables par la configuration. Redis était exactement ça : accessible via une variable d'environnement, prêt à être remplacé par une instance managée dans le cloud. La plomberie était faite. On ne l'utilisait juste pas pour le cache applicatif.

Certains services l'avaient même bien configuré pour des pools spécifiques. Le rate limiter dans le service d'authentification :

```yaml
pools:
    rate_limiter.cache:
        adapter: cache.adapter.redis
```

Ce qui explique l'incohérence qu'on avait vue en premier. Le *compteur* de rate limit allait dans Redis (partagé entre les pods). Le cache qui sous-tend la *vérification* du rate limit allait dans le filesystem (local au pod). Deux sources de vérité, l'une invisible à l'autre.

Le correctif : une ligne par service.

```yaml
framework:
    cache:
        app: cache.adapter.redis
        default_redis_provider: snc_redis.default
```

Treize fichiers. Treize modifications identiques. Le genre de fix qui te donne l'impression que tu aurais dû le voir plus tôt, sauf que c'est parfaitement invisible quand tu fais tourner une seule instance.

## Ce que deux pods débloquent vraiment

Le cache filesystem violait le facteur VI (les processus portent un état local qu'ils ne devraient pas) et le facteur VIII (tu ne peux pas scaler sans partager cet état). C'est le même problème vu sous deux angles : VI décrit ce qui ne va pas, VIII décrit ce qu'on ne peut pas faire à cause de ça.

Avec un backend de cache partagé, un deuxième pod est safe. Les deux pods construisent le même cache, voient les mêmes invalidations, s'accordent sur les mêmes rate limits. On peut ajouter un troisième pod sous la charge et le supprimer quand le trafic baisse. L'orchestrateur s'en charge ; l'application n'a pas besoin de le savoir.

Sans ça, le scaling horizontal est une source de bugs. Plus de pods = plus de divergences, plus de bugs "ça marche chez moi" impossibles à reproduire en local parce qu'en local il n'y a qu'un container.

Les sessions avaient le même problème — et potentiellement un pire. Douze des treize services utilisaient `session.storage.factory.native` — le filesystem. Un utilisateur dont la requête atterrit sur le pod A obtient une session liée au pod A. Sa prochaine requête va sur le pod B. Session perdue, il est déconnecté. Un seul service avait `RedisSessionHandler` configuré.

La mitigation partielle, c'est que la plupart de la plateforme tourne avec des APIs stateless basées sur JWT, donc l'usage des sessions est limité. Mais "limité" n'est pas "zéro". Les services qui créent des sessions — les flux d'authentification, l'état temporaire pendant les handshakes OAuth — ont un mode de défaillance visible par l'utilisateur qui attend le deuxième pod. Soit ces sessions migrent vers Redis, soit le code qui les crée est supprimé. Les laisser telles quelles, c'est une décision qui finira par se faire entendre.

## Ce qui fonctionnait déjà

Pour être juste avec le codebase : le Symfony Scheduler utilisait déjà Redis pour les verrous distribués :

```php
$schedule->lock($this->lockFactory->createLock('schedule_purge'));
```

Dans un environnement multi-pod, on ne veut pas que cinq instances lancent le même job de purge simultanément. Le verrou l'empêche. Redis rend le verrou visible entre les pods. Celui qui a écrit le scheduler savait exactement ce qu'il faisait.

Le même raisonnement ne s'était juste pas propagé à la configuration du cache — probablement parce que quand on tourne sur une seule instance, `cache.adapter.filesystem` est invisible. Ça marche, c'est rapide, ça demande zéro configuration. Le problème n'apparaît qu'à deux.

## Les quatre questions

Le facteur VI prend la plupart des applications par surprise lors d'une migration cloud. Pas parce que les développeurs ne connaissent pas les processus stateless — ils le savent en général — mais parce que le filesystem est toujours là, et le problème reste caché jusqu'à ce qu'on essaie de faire tourner une deuxième instance.

Avant de scaler un service Symfony horizontalement, quatre questions valent la peine d'être posées :

- Où va le cache applicatif ? (`cache.adapter.filesystem` doit devenir `cache.adapter.redis`)
- Où vont les sessions ? (`session.storage.factory.native` a besoin de Redis — ou supprimer les sessions entièrement si on est full JWT)
- Est-ce que quelque chose écrit dans `var/` à l'exécution qu'un autre pod aurait besoin de lire ?
- Y a-t-il quelque chose dans le chemin de code qui doit être mutuellement exclusif entre les pods ? (si oui, c'est un travail pour le <a href="https://symfony.com/doc/current/components/lock.html" target="_blank" rel="noopener noreferrer">composant Symfony Lock</a> backé par Redis, pas un mutex local)

Si les réponses pointent toutes vers des backing services partagés, tu es prêt. Si l'une d'elles pointe vers le filesystem local, la production trouvera le pod qui a construit son cache il y a trois heures et le servira à l'utilisateur qui s'y attend le moins.
