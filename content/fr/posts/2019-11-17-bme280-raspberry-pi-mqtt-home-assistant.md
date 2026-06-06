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

**La calibration Bosch n'est pas négociable.** J'ai commencé par lire le registre de température brute directement et le scaler naïvement. Le résultat était des nombres qui avaient l'air presque plausibles et qui étaient complètement faux. L'algorithme de compensation n'est pas une décoration optionnelle, c'est ce qui rend la sortie significative.

**Le polling bat les événements ici.** Le capteur ne pousse pas de données, on lui demande une lecture. Un cron job toutes les minutes est tout ce dont on a besoin pour surveiller une pièce. Le streaming en temps réel serait excessif et userait probablement le capteur plus vite.

**La découverte MQTT est sous-estimée.** Déclarer manuellement les capteurs dans `configuration.yaml` fonctionne, mais l'auto-découverte semble simplement juste. Publier un payload de config une fois, et Home Assistant s'en occupe. Ajouter un nouveau type de capteur plus tard prend environ trente secondes.

La pièce est maintenant à 21,4°C et 47% d'humidité. Je le sais sans rien ouvrir.

## Une note sur le SensorAPI officiel Bosch

En écrivant le driver, j'ai jeté un œil au <a href="https://github.com/boschsensortec/BME280_SensorAPI" target="_blank" rel="noopener noreferrer">SensorAPI officiel Bosch</a> pour référence. Deux choses ont retenu mon attention.

L'exemple userspace Linux ne fonctionne pas vraiment sur Raspberry Pi sans modifications : `ioctl` est appelé avant que `dev_addr` soit assigné, donc l'adresse du périphérique I²C n'est jamais correctement définie. Le correctif est évident une fois qu'on le voit, et plusieurs contributeurs ont buté sur le même bug indépendamment, mais ils attendaient en PR depuis des années. Certains attendent encore.

Il y a aussi la <a href="https://github.com/boschsensortec/BME280_SensorAPI/pull/94" target="_blank" rel="noopener noreferrer">PR #94</a> (toujours ouverte début 2025), signalant un comportement indéfini dans `bme280_get_sensor_mode()` : l'opérande gauche d'un `&` bit à bit est une variable non initialisée, détecté par analyse statique.

La puce elle-même est excellente. Mais le code de référence du fabricant est un point de départ, pas un évangile. Implémenter l'algorithme de compensation directement depuis la datasheet signifiait que je comprenais chaque ligne. Quand une lecture paraît bizarre, il n'y a pas de mystérieuse bibliothèque C à blâmer.

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
<a class="gh-card" href="https://github.com/guillaumedelre/bme280" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/bme280</span>
  </span>
  <span class="gh-card__desc">Driver Python pour le capteur BME280 — température, humidité et pression par I²C, avec publication MQTT et intégration Home Assistant.</span>
</a>
