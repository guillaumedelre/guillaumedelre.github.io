---
title: "Contrôler un lance-missiles USB en HTTP avec FastAPI et Docker"
aliases:
  - "/fr/2026/05/14/contrôler-un-lance-missiles-usb-en-http-avec-fastapi-et-docker/"
  - "/2026/05/14/controlling-a-usb-missile-launcher-over-http-with-fastapi-and-docker/"
  - "/fr/2017/02/21/contrôler-un-lance-missiles-usb-en-http-avec-fastapi-et-docker/"
  - "/2017/02/21/controlling-a-usb-missile-launcher-over-http-with-fastapi-and-docker/"
date: 2026-05-14
lastmod: 2026-11-18
tags: [python, docker]
format: experiment
featureimage: "https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg"
featureimagecaption: "Le Dream Cheeky Thunder, une petite tourelle USB qui tire des missiles en mousse"
images: ["https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg"]
description: "Comment on a branché un lance-missiles USB en mousse sur la CI — et ce que Docker, udev et WSL2 avaient à dire là-dessus."
---

{{< lead >}}
La règle était simple : celui qui casse le build de la CI offre le café à l'équipe. Ça a marché un moment. Puis quelqu'un a trouvé qu'il nous fallait un retour plus immédiat, quelque chose de physique, qui tire.
{{< /lead >}}

Un [Dream Cheeky Thunder][dct] a atterri sur un bureau peu après. Quatre missiles en mousse, un câble USB, et un consensus très clair dans l'équipe : le brancher au cluster, le relier au pipeline de build, et laisser la CI décider qui mérite une salve.

Le lanceur devait répondre à des appels HTTP depuis n'importe où sur le réseau, sans pilote ni interface graphique à installer, sans visée manuelle : un endpoint qui le fait tirer en direction du bureau du coupable.

Voici l'histoire de [dream-cheeky-thunder][repo].

## Un script de forum et une poignée d'octets

Dream Cheeky n'a jamais publié de spécification du protocole. Le lanceur parle USB HID brut, et le seul point de départ était un vieux script Python qui traînait dans des fils de forum. Vendor ID `0x2123`, product ID `0x1010`, et une poignée d'octets de contrôle que quelqu'un avait décodés par rétro-ingénierie des années auparavant.

C'était suffisant. Le protocole est simple : une séquence d'octets pour bouger les moteurs, une autre pour tirer. La difficulté, c'est que le lanceur n'a aucun retour de position : pas d'encodeurs, pas de fins de course en dehors des butées physiques aux extrémités, on le pilote donc à l'aveugle.

## De l'USB au HTTP

La CI devait déclencher le lanceur par le réseau. Un script local ne suffisait pas — le lanceur devait être accessible depuis n'importe quelle machine du cluster, y compris le serveur de build. Donc : une API REST.

FastAPI était le choix évident. Côté CI, la séquence de visée se résume à trois appels HTTP :

```bash
curl -X POST http://localhost:8000/park      # reset vers une position connue
curl -X POST http://localhost:8000/yaw/20    # rotation vers le bureau du coupable
curl -X POST "http://localhost:8000/fire?shots=2"
```

L'appel `/park` est plus important qu'il n'y paraît. Comme le lanceur n'a pas de retour de position, le serveur estime l'angle courant à partir du temps de rotation des moteurs. Cette estimation dérive. Un choc sur le matériel, une commande interrompue, l'imprécision même d'un suivi au chronomètre — tout s'accumule. Le parking pousse le lanceur à gauche, puis en bas, contre les butées physiques, en balayage complet à chaque fois, ce qui garantit l'alignement quoi que le serveur croie savoir ; sans lui, la visée reste approximative.

La référence complète de l'API est [dans le dépôt][api]. Il y a aussi une interface web si vous préférez cliquer plutôt que lancer `curl`.

## Docker ne connaît pas l'USB

Faire tourner ça dans un conteneur Docker sur le cluster, c'est là que les choses se sont corsées : les conteneurs ne voient pas les périphériques USB par défaut.

La directive `devices` dans `compose.yaml` expose le bus USB au conteneur :

```yaml
devices:
  - /dev/bus/usb:/dev/bus/usb
```

Pas suffisant. Première exécution : `USBError: [Errno 13] Access denied`. Le nœud de périphérique est bien là dans le conteneur, mais il hérite des permissions de l'hôte, et sur l'hôte seul root peut l'ouvrir par défaut.

La solution : une règle udev. Il suffit de déposer un fichier dans `/etc/udev/rules.d/` pour qu'udev l'applique au branchement du périphérique : mode `0666`, groupe `plugdev`. Le périphérique n'est plus réservé à root. La règle est fournie avec le projet, et les instructions d'installation sont [dans la doc][setup-linux].

## Quand WSL2 s'en est mêlé

La moitié de l'équipe est sous Windows avec Docker Desktop sur WSL2, ce qui ajoute une couche de plus entre le lanceur et le conteneur.

WSL2 n'a pas accès aux périphériques USB par défaut : c'est le noyau Windows qui les détient, et la directive `devices` seule ne sert à rien puisque WSL2 ne voit pas le matériel. La solution est [usbipd-win][usbipd-win], qui transfère le périphérique USB de Windows vers le noyau WSL2 par IP. Une fois ça fait, le chemin Linux fonctionne à l'identique : règle udev, directive `devices`, terminé.

L'attachement ne survit pas aux redémarrages, en revanche. Depuis la v4.2, usbipd peut partager automatiquement le périphérique grâce à une règle `policy`, et `usbipd attach --wsl --auto-attach` le rattache à chaque reconnexion. À eux deux, ils ont mis fin au mystère du « ça marchait hier » qui nous agaçait depuis des jours.

## Ce qui nous a surpris

Le positionnement au temps fonctionne suffisamment bien. Sans encodeurs, on s'attendait à un suivi d'angle quasi inutilisable ; en pratique, parquer le lanceur avant chaque séquence le gardait assez précis pour viser un bureau donné de façon fiable. Pas au millimètre, mais pour des missiles en mousse, ça suffit largement.

L'autre surprise, c'est l'erreur de permission. La directive `devices` est nécessaire mais pas suffisante, et l'erreur était déroutante justement parce que le périphérique était bien visible dans le conteneur. La règle udev est l'étape que la plupart des tutoriels passent sous silence. Quant à la règle du café, elle ne s'en est jamais remise : une fois le lanceur relié au pipeline, les builds cassés sont devenus beaucoup plus motivants à corriger.

{{< github repo="guillaumedelre/dream-cheeky-thunder" showThumbnail=false >}}

[dct]: https://web.archive.org/web/20171216061329/http://dreamcheeky.com/thunder-missile-launcher
[repo]: https://github.com/guillaumedelre/dream-cheeky-thunder
[api]: https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/api.md
[setup-linux]: https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/setup-linux.md
[usbipd-win]: https://github.com/dorssel/usbipd-win
