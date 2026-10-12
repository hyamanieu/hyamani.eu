---
title: Visualiser les mesures de capteurs IoT
author: hyamanieu
translationKey: iot-sensors-part-1
date: 2021-01-02T15:54:15+00:00
description: 'Connecter des capteurs domotiques à un rpi via le protocole zigbee. Enregistrer les données avec Node-Red. Visualiser les données live et historiques en utilisant les bibliothèques holoviz: Panel et Holoviews.'
slug: visualiser-les-mesures-de-capteurs-iot
featureimage: /img/2020/12/image_live_dashboard.jpeg
categories:
  - Panel
tags:
  - bokeh
  - holoviews
  - holoviz
  - IoT
  - mqtt
  - node-red
  - panel
  - postgres
  - zigbee

---
Ceci est la partie 1 d’une série de deux articles. [Voici le second](/fr/2021/01/10/visualiser-les-mesures-de-capteurs-iot-2-2/).

## Introduction

Cela fait deux ans que je n’ai pas écrit d’article. Il n’est jamais trop tard pour recommencer, surtout pour un sujet qui couvre toute la chaîne de la mesure physique à son analyse de manière simple.

On va donc parler d’un sujet qui me semble particulièrement important: comment enregistrer les mesures de capteurs qui envoient leurs données de manière asynchrone, visualiser leurs états en direct ou analyser leurs mesures passées.

Pour ce faire, on va parler de capteurs d’humidité, températures et pression atmosphérique dans le domaine de la domotique. L’objectif ne sera pas de réinventer Home Automation (HA), mais d’explorer **Node-Red** et les librairies **Holoviews** & **Panel** avec des capteurs à quelques euros. Le même principe pourrait être appliqué à tout domaine.

En quelques sections:

1. Hardware du *Zigbee* au *MQTT*
2. Visualiser les messages en direct avec *Holoviews stream*
3. Les archiver avec *Node-Red *et *PostgreSQL*
4. Visualiser les données archivées

Les sections 3 et 4 seront dans mon prochain post.

## Matériel nécessaire

- un ordinateur qui peut faire tourner [Zigbee2MQTT[EN]](https://www.zigbee2mqtt.io/getting_started/running_zigbee2mqtt.html), dans notre cas un **RaspberryPi 3B+**.
- un renifleur de réseau Zigbee sur USB, j’avais opté pour celui indiqué dans le guide de[ projetsdiy.fr: le CC2531 de Texas Instrument[FR]](https://projetsdiy.fr/test-pont-zigbee2mqtt-hack-gateway-xiaomi-smart-home-aqara-assemblage-systeme-alarme-node-red/) (les liens d’achat sont certainement sponsorisés), [d’autres sont compatibles[EN].](https://www.zigbee2mqtt.io/information/supported_adapters.html)
- un jeu de capteurs qui tourne sur Zigbee et de préférence [compatible avec la bibliothèque Zigbee2MQTT[EN]](https://www.zigbee2mqtt.io/information/supported_devices.html). J’ai un jeu de capteurs température, humidité, pression de Xiaomi.

Dans notre cas, le raspberryPi 3B+ servira autant de coordinateur **Zigbee**, que de serveur ou _broker_ **MQTT**, que de serveur pour **Node-Red**, **PostgreSQL** et de nos applications python **Panel**. Dans votre réseau local, vous pouvez évidemment séparer physiquement ces applications.

## Installation

Nous ne nous attarderons pas trop sur l’installation de chaque module puisque celles-ci sont déjà très bien documentées par chaque projet. Je propose une vue d’ensemble avec les détails pour ce projet.

### Renifleur Zigbee et broker MQTT

L’installation et la mise en place du renifleur est extrêmement bien expliqué directement sur le site https://www.zigbee2mqtt.io [EN] ou, en français, sur le site de projetsdiy.fr.

Petite astuce néanmoins: veillez à noter l’heure et l’ordre à laquelle vous enregistrez chaque capteur. Après vous être connecté au shell de votre raspberry pi, vous pouvez relire l’historique avec `journalctl -u zigbee2mqtt.service`, pour pouvoir donner des noms reconnaissables aux capteurs dans la liste sous `/opt/zigbee2mqtt/data/configuration.yaml`.

Vous devez aussi rajouter l’option `retain` pour chaque capteur: il faut faire en sorte que votre _broker_ MQTT enregistre la dernière donnée reçu par un capteur. En effet, zigbee2mqtt crée un canal de communication pour chaque capteur. Si aucun écouteur n’est sur le canal, l’information est perdue par défaut. Hors lors du démarrage de l’application, il serait intéressant d’avoir la dernière donnée plutôt qu’attendre qu’une nouvelle arrive qui, dans le cas de capteur _Zigbee_, peut prendre typiquement une heure s’il n’y a pas de changement brusque de température ou d’humidité.

> A noter: les capteurs Xiaomi ont un petit bouton qui permet d’envoyer immédiatement l’information.

Un extrait:

```
devices:
  '0x00158d00054692fd': 
    friendly_name: TH_Chambre
    retain: true
```

_TH_ pour _**T**empérature, **H**umidité_.

### Base de données

j’ai opté pour PostgreSQL que vous pourrez simplement installer sur Raspbian avec un `sudo apt install postgresql postgresql-contrib`

Je vous invite à faire un utilisateur _pi_ qui peut se connecter depuis une machine à distance, ainsi qu’une base de données qu’on appellera _sensors_. J’ai créé un schema _th_ dans lequel je ferai une table par capteur, nous y reviendrons plus tard avec **Node-Red**.

### Node-Red

La documentation est assez explicite: https://nodered.org/docs/getting-started/raspberrypi

### Python 3.7 et ses bibliothèques

Bien heureusement, la dernière version de Raspbian a la version 3.7 de Python de disponible sans effort. Un simple `sudo apt install python3` suffira (python2 reste l’application python par défaut).

Ensuite:

- `pip3 install paho-mqtt` pour MQTT.
- `pip3 install holoviews panel hvplot` pour le dashboard
- `pip3 install sqlalchemy psycopg2` pour accéder à la BDD.

Sur votre machine personnelle, je vous invite à utiliser un environnement virtuel en utilisant par exemple [conda][1], et à utiliser Jupyter Notebook. Vous pourrez en effet directement voir vos modifications sur vos dashboard _Panel_ et servir le notebook sur votre réseau local.

## Visualisez des données en direct

### Un peu de python…

vous pouvez reproduire le notebook jupyter ci-dessous, ou disponible [là](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/live_blog_fr.ipynb). Pour que ce soit lu depuis un ordinateur qui n’a pas jupyter installé, vous pouvez le convertir avec `jupyter nbconvert yournotebook.ipynb --to python`

### Un Dashboard à part entière

Pour en faire une application Panel, il vous faudra créer un layout Panel (avec l’exemple ci-dessus `layout=pn.panel(room_plots)`). Puis appeler en dernière instruction de la cellule de votre notebook:

```
layout.servable()
```

Enfin depuis le terminal de votre RPi (ou autre serveur ) entrez:

```
panel serve yournotebook.ipynb --port 8080 --address youraddress
```

Si vous voulez en faire une application accessible depuis votre réseau interne sur le port normal (80), il faut lier l’exécutif de python3.7 (la version de python utilisé ici, la plus haute stable sur raspberry pi 3B+ à ce jour):

```
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.7
```

> A noter que **toute** application python3.7 aura accès à tous les ports inférieurs à 1024, ce qui peut poser une faille de sécurité.

Vous pouvez retrouver [mon application intégrale sur github en suivant ce lien.](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/plotting_live_data.ipynb) Il vous faudra l’adapter à votre situation.

### En faire un service

Si vous voulez que l’application démarre seule depuis votre serveur Debian ou Raspberry pi, le mieux est de créer un service _systemd_. C’est très simple, il faut d’abord créer le fichier de configuration qui indique quoi lancer

```
>> sudo nano /etc/systemd/system/home_on_bokeh.service
```

```
Description=Serve panel to access zigbee sensor data
After=multi-user.target
 
[Service]
Type=simple
ExecStart=/home/pi/.local/bin/panel serve /home/pi/home_on_bokeh/plotting_live_data.py --port 80 --address yourrpiaddress
Restart=on-abort
User=pi
 
[Install]
WantedBy=multi-user.target
```

changez `yourrpiaddress` et, si nécessaire, `--port 80` pour un port accessible. Vous pouvez aussi définir les adresses du broker MQTT ainsi que le mot de passe avec la commande suivante:

```
>> sudo systemctl edit home_on_bokeh.service 
```

```
[Service]
Environment="MQTT_ADDRESS=localhost"
Environment="MQTT_USER=pi"
Environment="MQTT_PASSWORD=verycomplexpassphrase"
Environment="BOKEH_ALLOW_WS_ORIGIN=raspberrypi.local:80,192.168.10.10:80"
```

Ce fichier est accessible sous le dossier `/etc/systemd/system/home_on_bokeh.service.d` .

> A noter la variable `BOKEH_ALLOW_WS_ORIGIN`: sans elle, l’application n’est pas accessible sur le réseau local! Changer l’adresse par celle de votre serveur.

Si vous préférez que personne ne puisse lire son contenu (avec ses mots de passe!), allez voir [cette réponse sur *server fault*](https://serverfault.com/a/910655). Il ne vous reste plus qu’à démarrer le service:

```
>> sudo systemctl start home_on_bokeh
```

le laisser démarrer automatiquement:

```
>> sudo systemctl enable home_on_bokeh
```

vérifier que tout va bien (pas de `sudo` nécessaire)

```
>> systemctl status home_on_bokeh
```

Et voir ses logs:

```
>> journalctl -u home_on_bokeh -r
```

Il ne vous reste plus qu’à accéder à votre Dashboard depuis n’importe quelle machine sur le réseau local!

Dans le prochain post, on verra comment enregistrer ces données de manière permanente avec Node-Red puis en faire un dashboard (indice: c’est déjà sur GitHub).

[1]: https://docs.conda.io/en/latest/miniconda.html
