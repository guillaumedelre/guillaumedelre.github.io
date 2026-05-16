---
title: "D'un capteur à 10€ à un tableau de bord Home Assistant avec Raspberry Pi et MQTT"
date: 2019-11-17
categories: [iot]
tags: [raspberry-pi, bme280, mqtt, flask, i2c]
description: "Un capteur BME280 à 10€, un Raspberry Pi, et un broker MQTT : construire un moniteur de climat de pièce qui alimente Home Assistant."
---

La question était simple : quelle est la température et l'humidité dans mon bureau à domicile en ce moment ? Pas la météo dehors, pas une moyenne de ville — les conditions réelles dans la pièce où je passe la majeure partie de ma journée. Ouvrir une application météo pour ça semblait mal.

Un Raspberry Pi tournait déjà sur l'étagère. Un capteur BME280 coûte environ 10€. Ça aurait dû être un projet de week-end.

C'était globalement le cas, à l'exception de la partie où j'ai supposé que lire un capteur de température signifiait lire un registre.

## Quatre fils et une puce

Le Bosch BME280 mesure la température, l'humidité et la pression atmosphérique par I²C. Quatre fils vers les pins GPIO du Raspberry Pi, activer l'I²C dans `raspi-config`, et le capteur apparaît à l'adresse `0x77` sur le bus :

```bash
i2cdetect -y 1
```

C'est la partie facile. Le piège, c'est ce qui se passe ensuite.

## On ne lit pas juste la température

Le BME280 ne vous donne pas `21,5°C`. Il vous donne des valeurs ADC brutes : des entiers 20 bits qui ne signifient absolument rien par eux-mêmes. Pour obtenir une vraie température, il faut :

1. Lire les coefficients de calibration que Bosch a gravés dans l'EEPROM de la puce à l'usine (registres `0x88`, `0xA1`, `0xE1`)
2. Appliquer les formules de compensation Bosch : de l'arithmétique en virgule flottante double précision qui utilise ces coefficients pour transformer les valeurs brutes en vraies mesures
3. Attendre que la mesure soit terminée en scrutant le registre de statut

La compensation de température seule prend la valeur brute, applique une correction quadratique avec trois constantes de calibration, et crache une valeur en centièmes de degrés Celsius. La pression dépend de la température corrigée. L'humidité dépend des deux.

Tout est directement tiré de la <a href="https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bme280-ds002.pdf" target="_blank" rel="noopener noreferrer">datasheet Bosch</a>, rien d'inventé. Mais ça signifie que le driver n'est pas un programme de cinq lignes. C'est implémenter une spec, pas importer une bibliothèque.

## Le rendre accessible par le réseau

Une fois le driver fonctionnel, la question suivante était de savoir comment amener ces valeurs dans Home Assistant. Le chemin le plus simple : une API Flask avec deux endpoints.

`GET /bme280` retourne la lecture courante en JSON. `GET /bme280/publish` lit le capteur et pousse les trois valeurs vers un broker MQTT. Un cron job sur le Pi appelle l'endpoint publish toutes les quelques minutes, et Home Assistant récupère les valeurs en temps réel.

Le mécanisme de découverte MQTT a rendu la partie Home Assistant presque sans friction. Une commande `mosquitto_pub` par type de capteur — publier un payload JSON de config vers le bon topic — et les entités apparaissent automatiquement dans l'UI. Pas d'édition de `configuration.yaml`, pas de redémarrage requis.

```
BME280  ──I²C──►  bme280.py  ──►  Flask API  ──MQTT──►  Home Assistant
```

Le guide d'installation complet est <a href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">dans le repo</a>.

## Ce à quoi je ne m'attendais pas

:thermometer: **La calibration Bosch n'est pas négociable.** J'ai commencé par lire le registre de température brute directement et le scaler naïvement. Le résultat était des nombres qui avaient l'air presque plausibles et qui étaient complètement faux. L'algorithme de compensation n'est pas une décoration optionnelle, c'est ce qui rend la sortie significative.

:clock1: **Le polling bat les événements ici.** Le capteur ne pousse pas de données, on lui demande une lecture. Un cron job toutes les minutes est tout ce dont on a besoin pour surveiller une pièce. Le streaming en temps réel serait excessif et userait probablement le capteur plus vite.

:house: **La découverte MQTT est sous-estimée.** Déclarer manuellement les capteurs dans `configuration.yaml` fonctionne, mais l'auto-découverte semble simplement juste. Publier un payload de config une fois, et Home Assistant s'en occupe. Ajouter un nouveau type de capteur plus tard prend environ trente secondes.

La pièce est maintenant à 21,4°C et 47% d'humidité. Je le sais sans rien ouvrir.

## Une note sur le SensorAPI officiel Bosch

En écrivant le driver, j'ai jeté un œil au <a href="https://github.com/boschsensortec/BME280_SensorAPI" target="_blank" rel="noopener noreferrer">SensorAPI officiel Bosch</a> pour référence. Deux choses ont retenu mon attention.

L'exemple userspace Linux ne fonctionne pas vraiment sur Raspberry Pi sans modifications : `ioctl` est appelé avant que `dev_addr` soit assigné, donc l'adresse du périphérique I²C n'est jamais correctement définie. Le correctif est évident une fois qu'on le voit, et plusieurs contributeurs ont buté sur le même bug indépendamment, mais ils attendaient en PR depuis des années. Certains attendent encore.

Il y a aussi la <a href="https://github.com/boschsensortec/BME280_SensorAPI/pull/94" target="_blank" rel="noopener noreferrer">PR #94</a> (toujours ouverte début 2025), signalant un comportement indéfini dans `bme280_get_sensor_mode()` : l'opérande gauche d'un `&` bit à bit est une variable non initialisée, détecté par analyse statique.

La puce elle-même est excellente. Mais le code de référence du fabricant est un point de départ, pas un évangile. Implémenter l'algorithme de compensation directement depuis la datasheet signifiait que je comprenais chaque ligne. Quand une lecture paraît bizarre, il n'y a pas de mystérieuse bibliothèque C à blâmer.

<div style="border: 1px solid #e8e8e8; padding: 16px; margin-top: 2em; border-radius: 3px;">
  <img src="https://cdn.simpleicons.org/github" width="20" style="vertical-align: middle; margin-right: 8px;" />
  <strong><a href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">guillaumedelre/bme280</a></strong>
  <p style="margin: 8px 0 0; color: #828282; font-size: 14px;">Driver Python pour le capteur BME280 — température, humidité et pression par I²C, avec publication MQTT et intégration Home Assistant.</p>
</div>
