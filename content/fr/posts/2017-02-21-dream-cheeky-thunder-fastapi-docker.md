---
title: "Contrôler un lance-missiles USB en HTTP avec FastAPI et Docker"
date: 2017-02-21
categories: [iot]
tags: [fastapi, python, usb, docker, wsl2]
description: "Comment on a branché un lance-missiles USB en mousse sur le pipeline CI — et ce que Docker, udev et WSL2 avaient à dire là-dessus."
---

La règle était simple : celui qui casse le build CI offre le café à l'équipe. Ça a marché un moment. Puis quelqu'un a proposé qu'on ait un retour plus immédiat. Quelque chose de physique. Quelque chose qui tire.

Un <a href="http://www.dreamcheeky.com/thunder-missile-launcher" target="_blank" rel="noopener noreferrer">Dream Cheeky Thunder</a> a atterri sur un bureau peu après. Quatre missiles en mousse, un câble USB, et un consensus d'équipe très clair : le brancher au cluster, le câbler au pipeline de build, et laisser le CI décider qui mérite une volée.

Le lanceur devait répondre à des appels HTTP depuis n'importe où sur le réseau. Sans driver, sans GUI, sans visée manuelle. Juste un endpoint qui le fait tirer dans la direction du bureau du coupable.

Voilà l'histoire de <a href="https://github.com/guillaumedelre/dream-cheeky-thunder" target="_blank" rel="noopener noreferrer">dream-cheeky-thunder</a>.

![Dream Cheeky Thunder](https://raw.githubusercontent.com/guillaumedelre/dream-cheeky-thunder/develop/docs/Dream-Cheeky-Thunder.jpg)

## Pas de SDK, pas de docs, pas de problème

Dream Cheeky n'a jamais publié de spec de protocole. Le lanceur parle USB HID brut, et le seul point de départ était un script Python vendorisé de 2012 qui traînait dans des fils de forum. Vendor ID `0x2123`, product ID `0x1010`, et une poignée d'octets de contrôle que quelqu'un avait rétro-ingénié des années auparavant.

C'était suffisant. Le protocole est simple : envoyer une séquence d'octets pour bouger les moteurs, en envoyer une autre pour tirer. La partie délicate : le lanceur n'a aucun retour de position. Pas d'encodeurs, pas de fins de course en dehors des butées physiques aux extrémités. On le pilote à l'aveugle.

## Du USB au HTTP

Le pipeline CI devait déclencher le lanceur par le réseau. Un script local ne suffisait pas — le lanceur devait être accessible depuis n'importe quelle machine du cluster, y compris le serveur de build. Donc : une API REST.

FastAPI était le choix évident. Le flux de ciblage côté CI se résume à trois appels HTTP :

```bash
curl -X POST http://localhost:8000/park      # reset vers une position connue
curl -X POST http://localhost:8000/yaw/20    # rotation vers le bureau du coupable
curl -X POST "http://localhost:8000/fire?shots=2"
```

L'appel `/park` est plus important qu'il n'y paraît. Puisque le lanceur n'a pas de retour de position, le serveur estime l'angle courant en suivant le temps de rotation des moteurs. Cette estimation dérive. Un choc sur le hardware, une commande interrompue, ou simplement l'imprécision du tracking temporel — tout s'accumule. Le parking pousse les deux moteurs contre les butées physiques en balayage complet, ce qui garantit l'alignement quelle que soit la représentation interne du serveur. Sans ça, la visée est une approximation.

La référence complète de l'API est <a href="https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/api.md" target="_blank" rel="noopener noreferrer">dans le repo</a>. Il y a aussi une UI web si vous préférez cliquer plutôt que `curl`.

## Docker ne connaît pas l'USB

Faire tourner ça dans un conteneur Docker sur le cluster, c'est là que les choses ont commencé à devenir intéressantes : les conteneurs ne voient pas les périphériques USB par défaut.

Le mount `devices` dans `compose.yaml` expose le bus USB au conteneur :

```yaml
devices:
  - /dev/bus/usb:/dev/bus/usb
```

Pas suffisant. Première exécution : `USBError: [Errno 13] Access denied`. Le nœud de device est bien là dans le conteneur, mais il hérite des permissions du host, et sur le host seul root peut l'ouvrir par défaut.

La solution : une règle udev. Déposer un fichier dans `/etc/udev/rules.d/`, et le kernel applique le bon groupe et les bonnes permissions quand le device se branche. Après ça, l'utilisateur du conteneur peut l'ouvrir sans privilèges élevés. La règle est fournie avec le projet, les instructions d'installation sont <a href="https://github.com/guillaumedelre/dream-cheeky-thunder/blob/develop/docs/setup-linux.md" target="_blank" rel="noopener noreferrer">dans la doc</a>.

## WSL2 a rendu ça intéressant

La moitié de l'équipe tourne sous Windows avec Docker Desktop sur WSL2. C'est là que ça devient créatif.

WSL2 n'a pas accès aux périphériques USB par défaut : le kernel Windows les détient, et le mount `devices` seul ne fait rien parce que WSL2 ne voit simplement pas le hardware. La solution est <a href="https://github.com/dorssel/usbipd-win" target="_blank" rel="noopener noreferrer">usbipd-win</a>, qui transfère le périphérique USB de Windows vers le kernel WSL2 par IP. Une fois ça fait, le chemin Linux fonctionne à l'identique : règle udev, mount `devices`, terminé.

L'attachement ne survit pas aux redémarrages, cependant. usbipd v4+ a ajouté un mécanisme de policy qui automatise la reconnexion, ce qui a mis fin au mystère du "ça marchait hier" qui nous agaçait depuis des jours.

## Ce qui nous a vraiment surpris

**Le positionnement temporel fonctionne suffisamment bien.** Sans encodeurs, on s'attendait à ce que le tracking d'angle soit quasi-inutilisable. En pratique, le parking avant chaque séquence le maintenait assez précis pour viser un bureau spécifique de manière fiable. Pas au millimètre, mais la précision missile en mousse, ça convient.

**Le mount `devices` est nécessaire mais pas suffisant.** L'erreur de permission était déroutante précisément parce que le device était clairement visible dans le conteneur. La règle udev est la partie que la plupart des tutoriels passent discrètement sous silence.

**La règle café n'a plus jamais été la même après ça.** Une fois le lanceur câblé au pipeline, les builds cassés sont devenus beaucoup plus motivants à corriger.

<style>
.gh-card {
  display: block;
  border: 1px solid #d0d7de;
  padding: 16px;
  margin-top: 2em;
  border-radius: 6px;
  text-decoration: none !important;
}
.gh-card:hover { border-color: #8c959f; }
.gh-card,
.gh-card *,
.md-content .gh-card,
.md-content .gh-card * { text-decoration: none !important; }
.gh-card__head { display: flex; align-items: center; gap: 8px; }
.gh-card__head svg { flex-shrink: 0; fill: #1f2328 !important; }
.gh-card__repo { font-weight: 600; color: #1f2328 !important; }
.gh-card__desc { margin: 8px 0 0; color: #59636e !important; font-size: 14px; }
[data-theme=dark] .gh-card { border-color: #30363d; }
[data-theme=dark] .gh-card:hover { border-color: #6e7681; }
[data-theme=dark] .gh-card__head svg { fill: #e6edf3 !important; }
[data-theme=dark] .gh-card__repo { color: #e6edf3 !important; }
[data-theme=dark] .gh-card__desc { color: #8b949e !important; }
</style>
<a class="gh-card" href="https://github.com/guillaumedelre/dream-cheeky-thunder" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/dream-cheeky-thunder</span>
  </span>
  <span class="gh-card__desc">FastAPI + Docker + PyUSB — contrôle HTTP pour le lance-missiles USB Dream Cheeky Thunder. Pull requests bienvenus, surtout si vous avez une meilleure approche de calibration d'angle.</span>
</a>
