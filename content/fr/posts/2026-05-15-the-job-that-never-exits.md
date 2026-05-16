---
title: "Le Job Qui Ne Se Termine Jamais"
date: 2026-05-15T13:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 4
categories: [développement]
tags: [symfony, cloud, kubernetes, 12factor, messenger, scheduler]
description: "Symfony Scheduler versus Kubernetes CronJob : pas un choix entre deux outils, mais un choix sur l'endroit où la coordination doit vivre."
---

Le catalogue en listait dix. Des conteneurs aux noms comme `authentication_cron_purge`, `content_cron_youtube`, `sitemap_cron_generation`. Chacun exécutant le même genre de commande :

```bash
php bin/console messenger:consume scheduler_purge
```

Dans Docker Compose, ça fonctionne exactement comme ça en a l'air : un conteneur démarre, exécute `messenger:consume`, et reste vivant indéfiniment. Le Symfony Scheduler déclenche les tâches aux moments configurés, le consumer les récupère, elles tournent. Le conteneur ne se termine jamais sauf instruction contraire.

Déplacer ça vers Kubernetes soulève une question que Docker Compose ne posait jamais : où la coordination du scheduling doit-elle vivre ?

## Deux endroits pour mettre la même logique

Kubernetes a une réponse native pour le travail planifié : le CronJob. Il lit une expression cron depuis un manifest, crée un Pod au bon moment, et quand la commande se termine, le Pod se termine. Le planning vit dans l'infrastructure. Le conteneur est éphémère.

```yaml
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: purge
              command: ["php", "bin/console", "fmm:purge:refresh-token"]
          restartPolicy: OnFailure
```

Le Symfony Scheduler met la même logique en PHP. Le planning est une classe. L'expression cron est une string dans le code. La coordination se passe dans la couche applicative, pas dans le manifest d'infrastructure.

Ni l'une ni l'autre approche n'est fausse. Elles répondent à des questions différentes. Le choix concerne l'endroit où on veut que la coordination vive — et quelles garanties cet endroit peut offrir.

## Le lock qui change tout

Chaque classe de schedule dans la plateforme définit ses tâches dans le code :

```php
#[AsSchedule(name: 'purge')]
final readonly class PurgeSchedule implements ScheduleProviderInterface
{
    public function __construct(
        private LockFactory $lockFactory,
    ) {}

    public function getSchedule(): Schedule
    {
        $schedule = new Schedule();

        $schedule->add(RecurringMessage::cron('0 0 * * *',
            new RunCommandMessage('fmm:purge:refresh-token')
        ));
        $schedule->add(RecurringMessage::cron('0 0 * * *',
            new RunCommandMessage('fmm:purge:session-log --days=60')
        ));

        $schedule->lock($this->lockFactory->createLock('schedule_purge'));

        return $schedule;
    }
}
```

La dernière ligne est l'argument. `$schedule->lock()` acquiert un lock distribué Redis-backed avant de traiter tout message planifié. Si le service d'authentification fait tourner trois pods, les trois exécutent `messenger:consume scheduler_purge`. Les trois voient la tâche planifiée à minuit. Un seul acquiert le lock. Les deux autres le sautent.

C'est de la coordination dans la couche applicative — visible dans le code, testée avec le reste de l'application, soutenue par la même instance Redis déjà utilisée pour le cache et la gestion de session.

Les CronJobs Kubernetes poussent cette coordination dans la couche infrastructure, mais ne l'éliminent pas. Le contrôleur CronJob garantit une exécution *au moins une fois* — sous certaines conditions d'échec (redémarrages du contrôleur, décalage d'horloge, déclenchement concurrent), un CronJob peut se déclencher deux fois. L'application doit toujours gérer l'idempotence. Avec `LockFactory`, cette exigence est explicite et dans la même codebase que tout le reste.

## Ce que le Factor XII couvre vraiment

Le Factor XII de la <a href="https://12factor.net/admin-processes" target="_blank" rel="noopener noreferrer">méthodologie twelve-factor</a> concerne les processus admin : des tâches one-shot qui tournent dans le même environnement que l'application, puis se terminent. Migrations, corrections de données, réindexation manuelle. Le principe est que ces tâches utilisent la même image et la même config que l'application en cours d'exécution, pas un environnement séparé configuré à la main.

La plateforme suit ça pour les migrations de base de données : `doctrine:migrations:migrate` tourne dans l'entrypoint du conteneur, en utilisant la même image que l'application. Mêmes variables d'environnement, même connexion de base de données, même kernel Symfony. Quand ça se termine, l'application démarre.

Les tâches planifiées récurrentes sont une catégorie différente. Elles ne se terminent pas. Elles coordonnent entre instances. Elles sont plus proches de workers long-running que de commandes admin one-shot — et c'est exactement le modèle que le Symfony Scheduler utilise. La connexion au Factor XII est que les deux patterns tournent dans le même environnement que l'application. Là où ils diffèrent, c'est dans la forme : one-shot versus persistant.

## Le conteneur qui attend

Un conteneur `messenger:consume scheduler_purge` tournant à midi ne fait rien. Il écoute sur un transport Messenger, attendant que le scheduler émette un message à minuit. Ce message déclenchera la commande de purge. Puis il attendra à nouveau.

L'objection naturelle est le coût : dix conteneurs tournant continuellement pour se déclencher sur un planning, parfois une fois par jour, parfois plus souvent. Ce n'est pas du gaspillage au sens cloud-native — c'est le modèle. Un worker long-running qui coordonne via un lock partagé est le même pattern que n'importe quel consumer de queue. Le conteneur est inactif la plupart du temps. Le lock s'assure que quand le moment vient, exactement une instance le gère.

Ce que le setup actuel manque, c'est le signal à Kubernetes sur combien de temps attendre lors de l'arrêt. Les workers Symfony Messenger sont conçus pour gérer SIGTERM gracieusement — ils terminent le message courant avant de sortir, à condition que le handler lui-même ne bloque pas. Mais Kubernetes attend trente secondes par défaut avant d'envoyer SIGKILL. Une tâche de purge qui prend deux minutes serait interrompue. La correction appartient dans la spec du Deployment :

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 300  # correspondre à la durée de tâche la plus longue attendue
```

Définir ça à quelque chose de plus long que la durée du pire cas. Le consumer de scheduler termine son travail. Kubernetes attend. Aucun job n'est interrompu en cours d'exécution.

## Les dix conteneurs

À travers la plateforme, dix consumers de scheduler tournent dans Docker Compose et deviendraient des Deployments dans Kubernetes :

| Conteneur | Planning |
|---|---|
| `authentication_cron_purge` | minuit quotidien |
| `authentication_cron_ldap` | minuit + 2min quotidien |
| `bam_cron_purge` | minuit quotidien |
| `config_cron_redis` | piloté par env |
| `content_cron_applenews` | piloté par env |
| `content_cron_liveblog` | piloté par env |
| `content_cron_performance` | piloté par env |
| `content_cron_publication` | piloté par env |
| `content_cron_purge` | minuit quotidien |
| `content_cron_youtube` | piloté par env |

Six des dix utilisent des expressions cron pilotées par l'environnement, laissant le planning varier entre le staging et la production sans reconstruire l'image — une préoccupation Factor III qui vient gratuitement quand le planning vit dans la couche applicative.

Le plan de migration pour Kubernetes garde les dix en tant que Deployments. Chacun reçoit un `terminationGracePeriodSeconds` correspondant à sa durée du pire cas. Chacun a déjà le lock Redis. La coordination qui a fait survivre cette plateforme aux déploiements multi-pods dans Docker Compose est la même coordination qui la rendra correcte dans Kubernetes — parce qu'elle n'a jamais été dans l'infrastructure pour commencer.

C'est ça que le choix était vraiment.
