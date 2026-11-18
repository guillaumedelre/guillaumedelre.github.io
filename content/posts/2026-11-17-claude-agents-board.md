---
title: "Garder un œil sur plusieurs sessions Claude Code à la fois"
aliases:
  - "/fr/2026/11/17/garder-un-oeil-sur-plusieurs-sessions-claude-code-a-la-fois/"
  - "/2026/11/17/keeping-an-eye-on-several-claude-code-sessions-at-once/"
date: 2026-11-17
tags: [claude-code, homelab, docker]
format: showcase
featureimage: "images/claude-agents-board/home-tree.png"
featureimagecaption: "L'accueil : le nœud « Toi » en haut, les sessions ouvertes groupées par machine, les outils qu'elles utilisent en bas"
images: ["/images/claude-agents-board/home-tree.png"]
description: "Un petit tableau de bord auto-hébergé pour voir en temps réel ce que font toutes vos sessions Claude Code et leurs sous-agents."
lastmod: 2026-11-18
slug: "garder-un-oeil-sur-plusieurs-sessions-claude-code-a-la-fois"
---

{{< lead >}}
Une session Claude Code dans un terminal, ça se suit facilement. Trois sessions sur deux machines, chacune qui lance ses sous-agents, c'est une autre histoire. L'une restructure du code, l'autre fait tourner les tests, la troisième attend depuis vingt minutes que j'autorise une commande `psql`, et je ne m'en rends compte qu'en tombant par hasard sur le bon onglet.
{{< /lead >}}

C'est ce dernier cas, le vrai problème. Pas les agents qui travaillent : ceux qui *attendent*.

[claude-agents-board][repo] est le tableau de bord que je me suis écrit pour ça. Il tourne dans mon homelab, chaque session Claude Code du réseau local lui envoie ce qu'elle fait, et un seul onglet du navigateur montre qui travaille, qui est au repos et qui a besoin de moi. Je n'ai pas cherché d'outil existant avant de l'écrire : je suis parti directement des hooks de Claude Code.

## Comment il est branché

Les [hooks][hooks] de Claude Code se déclenchent sur à peu près tout ce que fait une session : un prompt, un appel d'outil, un sous-agent qui démarre ou s'arrête, une demande de permission, la fin d'un tour. Chaque machine envoie ces événements à un petit serveur Node, qui tient l'état et pousse les changements vers l'interface web. Il n'y a pas d'agent à installer, et rien ne change dans la façon d'utiliser Claude Code.

L'application sert elle-même son script d'installation. Brancher une machine tient en une ligne :

```bash
curl -fsSL http://agents.10.0.0.10.sslip.io/install-hooks.sh | sh -s -- --host portable
```

Le script ajoute les hooks à `~/.claude/settings.json` sans toucher à ceux qui existent déjà, active la télémétrie et s'intercale devant la status line existante. Une version PowerShell existe pour Windows.

Un détail à connaître : Claude Code sait appeler des hooks HTTP, mais dans mes tests il refusait de joindre une adresse privée comme `10.0.0.10` : seule l'adresse de bouclage passait. Les hooks sont donc des hooks `command` qui transmettent l'événement à `curl`. Ils sont synchrones, pour que les événements arrivent dans l'ordre, et abandonnent au bout de deux secondes si le tableau de bord ne répond pas. Un tableau de bord en panne ne bloque jamais un agent.

## Quand une session vous attend

La page d'accueil est un arbre : le nœud « Toi » en haut (c'est le libellé de l'interface), toutes les sessions ouvertes au milieu, groupées par machine, et en bas les outils qu'elles utilisent. Une session qui travaille respire en orange. Une session qui attend une réponse ou une permission passe en ambre, et le lien entre elle et vous se met à défiler.

Cet état ambre est la raison d'être de tout le reste. Un bandeau en haut de la page liste chaque session en attente, et un carillon, optionnel, sonne quand l'une d'elles demande une permission. Je ne découvre plus un agent bloqué par hasard.

{{< figure src="images/claude-agents-board/home-sessions.png" alt="Les boîtes des sessions : activité en cours et son chrono, barre de contexte, compactages et coût estimé" caption="Chaque boîte dit ce que fait la session et depuis combien de temps : un outil, « réfléchit », « t'attend » ou « inactive »." >}}

Chaque boîte indique aussi la barre de contexte, le nombre de compactages et le coût estimé. Les sous-agents en cours apparaissent en pastilles sur son bord. Une session inactive s'estompe progressivement entre cinq minutes et une heure, pour que celles que j'ai oubliées ne fassent pas concurrence à celles qui comptent.

## Quand un agent met trop de temps

{{< figure src="images/claude-agents-board/session-timeline.png" alt="Timeline d'une session : le fil principal, deux sous-agents et la liste de tâches" caption="La timeline d'une session : une ligne par tour du fil principal, par sous-agent et par tâche ; les pointillés donnent la fin estimée d'un agent en cours." >}}

Un clic sur une session ouvre son détail, et la timeline est la vue que j'utilise le plus. Pour un sous-agent en cours, des pointillés indiquent quand il devrait finir : c'est la durée médiane des agents du même type déjà terminés, sur le même projet quand il en compte au moins trois. Rien de malin, mais assez pour distinguer un agent qui travaille d'un agent qui coince. L'en-tête copie aussi `claude --resume <id>` d'un clic, pour reprendre la session depuis un autre terminal.

## Quand le contexte se remplit

Quand une session devient lente et un peu vague, la première question est toujours « elle va compacter ? », et la réponse est dans un terminal que je ne regarde pas.

Les hooks ne donnent pas la taille du contexte, mais la télémétrie [OpenTelemetry][otel] de Claude Code permet de la calculer : chaque requête au modèle indique ses tokens d'entrée et de cache. Encore faut-il écarter les requêtes qui n'appartiennent pas à la conversation (titres de session, résumés de compactage) et séparer celles des sous-agents, identifiables depuis Claude Code 2.1.310. Le tableau de bord suit donc un contexte par fil ; le [README][repo] détaille le calcul.

{{< figure src="images/claude-agents-board/session-map.png" alt="Carte d'une session : la jauge de contexte entoure l'agent principal" caption="Dans la carte d'une session, la jauge de contexte entoure l'agent principal ; elle passe à l'orange à 60 % et au rouge à 85 %." >}}

## Combien il reste de la semaine

Avec un abonnement, le chiffre qui compte, c'est ce qu'il reste de la semaine. Ni les hooks ni la télémétrie ne le transmettent : seule la [status line][statusline] le reçoit, pour les abonnements Pro, Max et Ultra. C'est pour ça que le script d'installation s'intercale devant elle. Il envoie ces valeurs au tableau de bord, puis lance la status line d'origine, sans rien changer dans le terminal.

{{< figure src="images/claude-agents-board/limits.png" alt="Les limites de l'abonnement dans la barre d'état : semaine et fenêtre de 5 h avec leurs dates de remise à zéro" caption="Les deux limites de l'abonnement, la semaine et la fenêtre de 5 h, dans la barre du bas ; la date de remise à zéro s'affiche au survol." >}}

La télémétrie donne aussi les tokens et un coût estimé, par session et sur sept jours par modèle. Ce coût est celui du tarif public de l'API : avec un abonnement, ce n'est pas ce qui est facturé. Je le lis comme un ordre de grandeur, et comme un moyen de repérer la session qui consomme de l'Opus depuis tout l'après-midi.

## Et pour le plaisir

{{< figure src="images/claude-agents-board/session-office.png" alt="Le bureau : l'agent principal à l'accueil, un sous-agent à sa table" caption="La vue bureau : l'agent principal à l'accueil, chaque sous-agent à sa table." >}}

Chaque session a aussi son bureau en pixel art. Les sous-agents entrent par la porte, s'installent à une table, tapent ou lisent selon l'outil qu'ils utilisent, puis repartent. C'est utile ? Pas vraiment, la timeline dit la même chose en plus précis. Mais c'est la vue la plus amusante à regarder. Les personnages viennent des packs [MetroCity][metrocity] et [MetroCity Interior][metrocity-interior] de JIK-A-4, sous licence CC0.

## Ce qu'il ne fait pas

{{< alert >}}
Il est pensé pour un réseau local, et ça se voit. L'interface n'a pas d'authentification : n'importe quel appareil du réseau peut lire les prompts, les commandes et les fichiers lus par les agents. Un jeton optionnel protège les routes qui reçoivent les données, pas la page ni son API, qui permet aussi de renommer ou d'effacer des sessions. Je ne l'exposerais pas en dehors de la maison.
{{< /alert >}}

Les hooks ne remontent pas tout non plus. Les messages intermédiaires du modèle ne sortent jamais de Claude Code, et les sous-agents lancés par un workflow arrivent sans leur consigne. Les tokens et le coût sont comptés par session, sans répartition par sous-agent. Sous Windows, le script PowerShell ne s'intercale pas encore devant la status line : il faut la brancher à la main. Et une session dont le terminal a été fermé brutalement reste affichée jusqu'au nettoyage automatique, au bout d'un jour par défaut.

## L'essayer

Il vous servira si vous faites tourner plusieurs sessions Claude Code en parallèle, sur une ou plusieurs machines d'un même réseau local. Il faut Docker sur la machine qui l'héberge, puis :

```bash
git clone https://github.com/guillaumedelre/claude-agents-board.git && cd claude-agents-board
cp .env.example .env    # IP et DOMAIN de la machine, jeton optionnel
docker compose up -d --build
```

Chaque machine se branche ensuite avec la commande `curl` donnée plus haut. Chez moi, il tourne comme une stack de plus dans [le homelab]({{< relref "2026-05-17-homelab-docker-compose-traefik.md" >}}), derrière Traefik, sur un nom en [sslip.io]({{< relref "2025-04-17-traefik-local-https-mkcert-sslip.md" >}}). Les captures de cet article viennent d'une instance de démonstration alimentée par des sessions fictives.

Si quelque chose ne marche pas chez vous, ou s'il vous manque quelque chose, les issues et les pull requests sont ouvertes.

{{< github repo="guillaumedelre/claude-agents-board" >}}

[repo]: https://github.com/guillaumedelre/claude-agents-board
[hooks]: https://code.claude.com/docs/en/hooks
[otel]: https://code.claude.com/docs/en/monitoring-usage
[statusline]: https://code.claude.com/docs/en/statusline
[metrocity]: https://jik-a-4.itch.io/metrocity-free-topdown-character-pack
[metrocity-interior]: https://jik-a-4.itch.io/metrocity
