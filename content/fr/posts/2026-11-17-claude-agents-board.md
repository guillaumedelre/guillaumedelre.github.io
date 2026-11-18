---
title: "Garder un œil sur plusieurs sessions Claude Code à la fois"
date: 2026-11-17
tags: [claude-code, homelab, docker]
format: showcase
images: ["/images/claude-agents-board/home-tree.png"]
description: "Un petit tableau de bord auto-hébergé pour voir en temps réel ce que font toutes vos sessions Claude Code et leurs sous-agents."
translationKey: "claude-agents-board"
lastmod: 2026-11-17
slug: "garder-un-oeil-sur-plusieurs-sessions-claude-code-a-la-fois"
---

Une session Claude Code dans un terminal, ça se suit facilement. Trois sessions sur deux machines, chacune qui lance ses sous-agents, c'est une autre histoire. L'une refactore, l'autre fait tourner les tests, la troisième attend depuis vingt minutes que j'autorise une commande `psql`, et je ne m'en rends compte qu'en tombant par hasard sur le bon onglet.

C'est ce dernier cas, le vrai problème. Pas les agents qui travaillent : ceux qui *attendent*.

Alors je me suis fait un petit tableau de bord : [claude-agents-board][repo]. Il tourne dans mon homelab, chaque session Claude Code du réseau local lui envoie ce qu'elle fait, et un seul onglet du navigateur montre qui travaille, qui est au repos et qui a besoin de moi. Ce n'est pas un outil révolutionnaire. C'est un outil pratique.

## L'idée tient en une phrase

Les [hooks][hooks] de Claude Code se déclenchent sur à peu près tout ce que fait une session : un prompt, un appel d'outil, un sous-agent qui démarre ou s'arrête, une demande de permission, la fin d'un tour. Chaque machine envoie ces événements à un petit serveur Node, qui tient l'état et pousse les changements vers une interface web en server-sent events. C'est tout. Pas d'agent à installer, rien à changer dans sa façon d'utiliser Claude Code.

Un petit détail à connaître quand même. Claude Code sait faire des hooks HTTP, mais dans mes tests il a refusé d'appeler une adresse privée comme `10.0.0.10` : seul le loopback passait. Les hooks sont donc de simples hooks `command` qui envoient le JSON de l'événement à `curl`. Ils sont synchrones, pour que les événements arrivent dans l'ordre, et abandonnent au bout de deux secondes si le tableau de bord ne répond pas. Un tableau de bord en panne ne bloque jamais un agent.

L'application sert elle-même son script d'installation, donc brancher une machine tient en une ligne :

```bash
curl -fsSL http://agents.10.0.0.10.sslip.io/install-hooks.sh | sh -s -- --host portable
```

Le script ajoute les hooks dans `~/.claude/settings.json` sans toucher à ceux qui existent déjà, active la télémétrie et enveloppe la status line (j'y reviens plus bas). Une version PowerShell existe pour Windows. Le reste de l'installation est dans le [README][repo].

## Une page, toutes les sessions

![Accueil : toi en haut, les sessions groupées par machine au milieu, les stations partagées en bas](/images/claude-agents-board/home-tree.png)

La page d'accueil est un arbre. Toi en haut, toutes les sessions ouvertes au milieu, groupées par machine, et en bas les « stations » qu'elles utilisent : modèle, fichiers, terminal, réseau, agents, etc. Chaque appel d'outil envoie une impulsion de la session vers la station, puis une autre au retour. Une session qui travaille respire en orange ; une session qui t'attend passe en ambre, et le lien entre elle et toi se met à défiler.

![Les boîtes des sessions : activité en cours et son chrono, barre de contexte, compactages et coût estimé](/images/claude-agents-board/home-sessions.png)

Chaque boîte en dit un peu plus que « occupée » ou « pas occupée ». Sous le titre de la session et le nom du projet : ce qu'elle fait en ce moment et depuis combien de temps (un outil, « réfléchit », « t'attend », « inactive »), une barre de contexte avec son pourcentage, le nombre de compactages et le coût estimé. Les sous-agents en cours apparaissent en petites pastilles sur le bord haut, chacun avec la jauge de son propre contexte. Une session inactive s'estompe doucement entre cinq minutes et une heure, pour que celles que j'ai oubliées ne fassent plus concurrence à celles qui comptent.

Cet état ambre, c'est la raison d'être de tout le reste. Un bandeau en haut liste chaque session qui attend une réponse ou une permission, et un petit carillon, optionnel, sonne quand l'une d'elles en demande une. Je ne découvre plus un agent bloqué par hasard.

La barre latérale range les sessions par machine, puis par dossier de projet. Les sessions qui tournent dans un worktree créé par Claude Code (sous `.claude/worktrees/`) sont rattachées à leur dépôt au lieu d'apparaître comme des inconnues.

## Ce que fait vraiment une session

![Timeline d'une session : le fil principal, deux sous-agents et la liste de tâches](/images/claude-agents-board/session-timeline.png)

Un clic sur une session ouvre son détail, et la timeline est la vue que j'utilise le plus. Une ligne pour le fil principal avec une barre par tour, une ligne par sous-agent de son lancement à sa fin, une ligne par tâche de la liste de tâches de Claude, et les appels d'outils en petites marques en dessous.

Pour un agent en cours, des pointillés indiquent quand il devrait finir. L'estimation, c'est simplement la durée médiane des agents du même type déjà terminés, sur le même projet dès qu'il en compte trois, tous projets confondus sinon. Rien de malin, mais assez pour distinguer « normal » de « il y a un truc qui coince ». L'en-tête de la session copie aussi `claude --resume <id>` d'un clic, pratique pour reprendre une session depuis un autre terminal.

La vue journal liste les prompts et les appels d'outils dans l'ordre, et chaque tour se termine par la réponse finale du modèle. C'est tout ce que donnent les hooks : les messages intermédiaires ne sortent jamais de Claude Code.

## Combien ça coûte, à peu près

![Tokens et coût estimé sur 7 jours, empilés par modèle](/images/claude-agents-board/home-tokens.png)

Les hooks ne transmettent pas la consommation de tokens, mais Claude Code sait exporter des événements [OpenTelemetry][otel], un par requête au modèle. Le script d'installation les active et les envoie au tableau de bord. Chaque session a un petit badge avec ses tokens et son coût estimé, et l'accueil affiche les sept derniers jours par modèle, avec la part lue depuis le cache.

Le coût, c'est l'estimation de Claude Code au tarif public de l'API. Avec un abonnement, ce n'est pas ce qui est facturé. Je le lis comme un ordre de grandeur, et comme un moyen de repérer la session qui consomme de l'Opus depuis tout l'après-midi.

## Quand le contexte se remplit

Le contexte, c'est le chiffre qui me manquait le plus. Quand une session devient lente et un peu vague, la première question est toujours « elle va compacter ? », et la réponse est dans un terminal que je ne regarde pas.

Les hooks ne le disent pas. La télémétrie, si, indirectement : chaque requête au modèle donne ses tokens d'entrée, de cache lu et de cache écrit, et leur somme, c'est la taille du contexte envoyé avec cette requête. Le piège, c'est que toutes les requêtes n'appartiennent pas à la conversation. Claude Code demande aussi au modèle des titres de session, des résumés de compactage ou des vérifications du mode auto, et les sous-agents font leurs propres requêtes dans la même session. Chaque événement porte un `query_source`, et depuis Claude Code 2.1.310 les requêtes des sous-agents portent aussi un `agent_id`. Le tableau de bord garde donc un contexte par fil : le principal, et un par sous-agent. Les requêtes annexes sont ignorées.

![Carte d'une session : la jauge de contexte entoure l'agent principal](/images/claude-agents-board/session-map.png)

Le pourcentage est calculé sur la fenêtre du modèle : 200 k pour les modèles 3.x à 4.6 sans suffixe `[1m]`, 1 M pour les autres. La barre passe à l'orange à 60 % et au rouge à 85 %, et chaque compactage (hook `PostCompact`) ajoute un au compteur `⟲`. Dans la vue carte d'une session, la même jauge entoure l'agent principal.

### Et la limite de la semaine

Avec un abonnement, le chiffre qui compte vraiment n'est pas le coût, c'est ce qu'il reste de la semaine. Ni les hooks ni la télémétrie ne le transmettent. Seule la [status line][statusline] le reçoit : un objet `rate_limits` avec les fenêtres de 7 jours et de 5 heures, en pourcentage utilisé avec la date de reset, pour les abonnements Pro, Max et Ultra, après la première réponse du modèle.

Le script d'installation enveloppe donc la status line déjà configurée, quelle qu'elle soit. Un petit script généré envoie l'entrée de la status line au tableau de bord, puis lance la commande d'origine, mise de côté et remise en place par `--uninstall`. Rien ne change dans le terminal. Au passage, la taille exacte de la fenêtre de contexte arrive aussi, et remplace la règle 200 k / 1 M pour le fil principal.

![Les limites de l'abonnement dans la barre d'état : semaine et fenêtre de 5 h avec leurs dates de reset](/images/claude-agents-board/limits.png)

Les deux limites s'affichent dans la barre du bas, avec les dates de reset au survol. Quand une fenêtre dépasse sa date de reset, elle repart à zéro en attendant la prochaine valeur.

## La partie juste pour le plaisir

![Le bureau : l'agent principal à l'accueil, un sous-agent à sa table](/images/claude-agents-board/session-office.png)

Chaque session a aussi son bureau, en pixel art. L'agent principal est derrière le comptoir d'accueil, en veste orange ; chaque sous-agent entre par la porte, s'installe à une table, tape ou lit selon l'outil qu'il utilise, puis se lève et repart quand il a fini. Quand l'agent principal est au repos depuis un moment, il va prendre un café ou s'installe sur le canapé devant la télé. Une session qui ne donne plus de nouvelles fait la sieste.

C'est utile ? Pas vraiment. La timeline dit la même chose, en plus précis. Mais c'est de loin la plus amusante à regarder. Les sprites viennent des packs [MetroCity][metrocity] et [MetroCity Interior][metrocity-interior] de JIK-A-4, sous licence CC0.

## Ce qu'il ne fait pas

Il est pensé pour un réseau local, et ça se voit. L'interface n'a pas d'authentification : n'importe quel appareil du réseau peut lire les prompts, les commandes et les fichiers lus par les agents. Un token optionnel protège les routes qui reçoivent les données (hooks, télémétrie, status line), pas la page ni son API, qui permet aussi de renommer ou d'effacer des sessions. Je ne l'exposerais pas en dehors de la maison.

Les hooks ne remontent pas tout non plus. Pas de messages intermédiaires du modèle, et les sous-agents lancés par un workflow arrivent sans leur consigne. Les tokens et le coût sont comptés par session, sans répartition par sous-agent ; seul le contexte est suivi par fil. Le script PowerShell n'enveloppe pas encore la status line : sous Windows, il faut la brancher à la main (le README explique comment). Une session dont le terminal a été fermé sans sortie propre reste « au repos », ou passe à « Sans nouvelles » au bout de dix minutes si elle était en plein tour, jusqu'à ce que le nettoyage automatique la retire (au bout d'un jour par défaut).

Rien de tout ça ne me gêne pour ce qu'il est censé être : un onglet sur lequel je jette un œil pour savoir si quelque chose a besoin de moi. Il tourne comme un stack de plus à côté des autres dans le [homelab]({{< relref "2026-05-17-homelab-docker-compose-traefik.md" >}}), derrière Traefik, sur un nom en [sslip.io]({{< relref "2025-04-17-traefik-local-https-mkcert-sslip.md" >}}). Les captures ci-dessus viennent d'une instance de démonstration alimentée par des sessions fictives.

<a class="gh-card" href="https://github.com/guillaumedelre/claude-agents-board" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/claude-agents-board</span>
  </span>
  <span class="gh-card__desc">Tableau de bord web, en temps réel, des sessions Claude Code et de leurs sous-agents, alimenté par les hooks. Docker Compose, Node, React. Licence MIT.</span>
</a>

[repo]: https://github.com/guillaumedelre/claude-agents-board
[hooks]: https://code.claude.com/docs/en/hooks
[otel]: https://code.claude.com/docs/en/monitoring-usage
[statusline]: https://code.claude.com/docs/en/statusline
[metrocity]: https://jik-a-4.itch.io/metrocity-free-topdown-character-pack
[metrocity-interior]: https://jik-a-4.itch.io/metrocity
