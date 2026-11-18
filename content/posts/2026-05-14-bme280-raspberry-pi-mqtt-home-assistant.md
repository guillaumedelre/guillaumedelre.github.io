---
title: "D'un capteur à 10 € à un tableau de bord Home Assistant avec Raspberry Pi et MQTT"
aliases:
  - "/fr/2026/05/14/dun-capteur-à-10-à-un-tableau-de-bord-home-assistant-avec-raspberry-pi-et-mqtt/"
  - "/2026/05/14/from-a-10-sensor-to-a-home-assistant-dashboard-with-a-raspberry-pi-and-mqtt/"
  - "/fr/2019/11/17/dun-capteur-à-10-à-un-tableau-de-bord-home-assistant-avec-raspberry-pi-et-mqtt/"
  - "/2019/11/17/from-a-10-sensor-to-a-home-assistant-dashboard-with-a-raspberry-pi-and-mqtt/"
date: 2026-05-14
lastmod: 2026-11-18
tags: [python, raspberry-pi, home-assistant]
format: experiment
description: "Un capteur BME280 à 10 €, un Raspberry Pi et un broker MQTT : une sonde de température et d'humidité pour une pièce, qui alimente Home Assistant."
---

{{< lead >}}
La question était simple : quelles sont la température et l'humidité dans mon bureau à domicile en ce moment ? Pas la météo dehors, pas une moyenne de ville — les conditions réelles dans la pièce où je passe la majeure partie de ma journée. Ouvrir une appli météo pour ça me paraissait absurde.
{{< /lead >}}

Un Raspberry Pi tournait déjà sur l'étagère. Un capteur BME280 coûte environ 10 €. Ça aurait dû être un projet de week-end.

C'était globalement le cas, à l'exception de la partie où j'ai supposé que lire un capteur de température signifiait lire un registre.

## Quatre fils et une puce

Le Bosch BME280 mesure la température, l'humidité et la pression atmosphérique par I²C. Quatre fils vers les broches GPIO du Raspberry Pi, activer l'I²C dans `raspi-config`, et le capteur apparaît sur le bus, à `0x77` sur ma carte (`0x76` sur beaucoup de modules, selon le câblage de la broche SDO) :

```bash
i2cdetect -y 1
```

C'est la partie facile. Le piège, c'est ce qui se passe ensuite.

## On ne lit pas juste la température

Le BME280 ne vous donne pas `21,5°C`. Il vous donne des valeurs ADC brutes : des entiers 20 bits pour la température et la pression, 16 bits pour l'humidité, qui ne signifient rien par eux-mêmes. Pour obtenir une vraie température, il faut :

1. Lire les coefficients de calibration que Bosch a gravés dans la mémoire non volatile de la puce à l'usine (registres `0x88`, `0xA1`, `0xE1`)
2. Appliquer les formules de compensation Bosch, qui utilisent ces coefficients pour transformer les valeurs brutes en vraies mesures : de l'arithmétique entière pour la température, de la virgule flottante double précision pour la pression et l'humidité
3. Attendre que la mesure soit terminée en scrutant le registre de statut

La compensation de température seule prend la valeur brute, applique une correction quadratique avec trois constantes de calibration, et crache une valeur en centièmes de degrés Celsius. La pression dépend de la température corrigée, l'humidité aussi.

Tout est directement tiré de la [datasheet Bosch][datasheet], rien de malin. Je ne suis pas parti de zéro non plus : le driver a commencé comme le script de Matt Hawkins publié sur [Raspberry Pi Spy][rpi-spy], une transcription directe de ces formules. Du coup, le driver ne tient pas en cinq lignes : c'est une transcription de la spec.

## Sortir les mesures du Pi

Une fois le driver fonctionnel, la question suivante était de savoir comment amener ces valeurs dans Home Assistant. Le chemin le plus simple : une API Flask avec deux endpoints.

`GET /bme280` retourne la lecture courante en JSON. `GET /bme280/publish` lit le capteur et pousse les trois valeurs vers un broker MQTT. Une tâche cron sur le Pi appelle l'endpoint publish toutes les minutes, et Home Assistant récupère les valeurs dès leur publication.

Le mécanisme de découverte MQTT a rendu la partie Home Assistant presque sans friction. Une commande `mosquitto_pub` par type de capteur — publier une charge utile JSON de configuration vers le bon topic — et les entités apparaissent automatiquement dans l'interface. Pas d'édition de `configuration.yaml`, pas de redémarrage requis.

{{< mermaid >}}
flowchart LR
  S[BME280] -- I²C --> D[bme280.py] --> A[Flask API] -- MQTT --> H[Home Assistant]
{{< /mermaid >}}

Le guide d'installation complet est [dans le repo][repo].

## Une note sur le SensorAPI officiel Bosch

En travaillant sur le driver, j'ai jeté un œil au [SensorAPI officiel Bosch][sensorapi] à titre de référence. Deux choses ont retenu mon attention.

Pendant des années, l'exemple userspace Linux n'a pas fonctionné sur Raspberry Pi sans modifications. Plusieurs contributeurs ont buté indépendamment sur le même bug : `ioctl` était appelé avant que `dev_addr` soit assigné, donc l'adresse du périphérique I²C n'était jamais correctement définie. Le correctif est évident une fois qu'on le voit, et des PR le décrivaient dès 2021, mais il n'a été intégré qu'en mai 2023. Une de ces anciennes PR est toujours ouverte.

Il y a aussi la [PR #94][pr94] (ouverte en janvier 2025 et toujours ouverte), signalant un comportement indéfini dans `bme280_get_sensor_mode()` : l'opérande gauche d'un `&` bit à bit est une variable non initialisée, détecté par analyse statique.

La puce elle-même est excellente. Mais le code de référence du fabricant est un point de départ, pas parole d'évangile. Reprendre l'algorithme de compensation ligne par ligne avec la datasheet m'a obligé à comprendre chaque ligne. Quand une lecture paraît bizarre, il n'y a pas de mystérieuse bibliothèque C à blâmer.

## Ce à quoi je ne m'attendais pas

La calibration Bosch n'est pas négociable. J'ai commencé par lire le registre de température brut directement et le mettre à l'échelle naïvement. Le résultat : des nombres qui avaient l'air presque plausibles et qui étaient complètement faux. Sans l'algorithme de compensation, la sortie ne veut rien dire.

Pour ce projet, mieux vaut interroger le capteur qu'attendre des événements : il ne pousse pas de données, on lui demande une lecture. Une tâche cron toutes les minutes suffit pour surveiller une pièce. Mesurer beaucoup plus souvent serait excessif, et chaufferait assez la puce pour fausser la température.

Quant à la découverte MQTT, je la trouve sous-estimée. Déclarer manuellement les capteurs dans `configuration.yaml` fonctionne, mais l'auto-découverte tombe sous le sens : on publie une charge utile de configuration une fois, et Home Assistant s'en occupe. Ajouter un nouveau type de capteur plus tard prend environ trente secondes.

La pièce est maintenant à 21,4 °C et 47 % d'humidité. Je le sais sans rien ouvrir.

{{< github repo="guillaumedelre/bme280" >}}

[datasheet]: https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bme280-ds002.pdf
[rpi-spy]: https://www.raspberrypi-spy.co.uk/
[repo]: https://github.com/guillaumedelre/bme280
[sensorapi]: https://github.com/boschsensortec/BME280_SensorAPI
[pr94]: https://github.com/boschsensortec/BME280_SensorAPI/pull/94
