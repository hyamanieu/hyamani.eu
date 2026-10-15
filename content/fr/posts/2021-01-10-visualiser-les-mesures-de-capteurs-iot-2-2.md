---
title: Visualiser les mesures de capteurs IoT, 2/2
author: hyamanieu
translationKey: iot-sensors-part-2
date: 2021-01-10T18:48:00+00:00
description: Nous allons enregistrer les données qui arrivent sur le broker en utilisant Node-Red sur un serveur PostgreSQL, puis les visualiser avec les packages Holoviz.
slug: visualiser-les-mesures-de-capteurs-iot-2-2
featureimage: /img/2021/01/image.png
categories:
  - Panel
tags:
  - holoviews
  - mqtt
  - node-red
  - panel
  - postgresql
series: ['iot-sensors']

---
_Cet article est la suite de [la partie 1](/fr/2021/01/02/visualiser-les-mesures-de-capteurs-iot/)._

## Node-Red: Enregistrer les données

### Récupérer les messages MQTT…

Comme introduit dans la partie précédente, nous allons enregistrer les données qui arrivent sur le broker en utilisant [Node-Red](/fr/2021/01/02/visualiser-les-mesures-de-capteurs-iot/#node-red) sur un serveur [PostgreSQL](/fr/2021/01/02/visualiser-les-mesures-de-capteurs-iot/#base-de-données).

Si l’installation a réussie, Node-Red devrait être accessible sur le port 1880 (par exemple `raspberrypi.local:1880`). Il vous faudra installer depuis celui-ci:

- [Les nodes postgresql de Digitaloak](https://flows.nodered.org/node/@digitaloak/node-red-contrib-digitaloak-postgresql)

Et c’est tout!

Un node MQTT existe dans l’installation standard. En plaçant un node, ou nœud, _MQTT IN_ sur votre feuille de flux (ou _flow_), vous pouvez configurer le serveur ou _broker_ comme on l’a fait en python la dernière fois, puis le nœud afin d’écouter un canal en particulier.

Node Red fonctionne aussi avec des _scopes_. Il y en a trois:

- **msg**: ce *scope* contient les objets partagés entre les nœuds liés entre eux. Celui qui nous intéresse est le message porté par votre source, qui se trouve sous `msg.payload`.
- **flow**: ce *scope* contient tous les objets que vous désirez partager avec l’ensemble des nœuds présent sur une feuille.
- **global**: comme son nom l’indique, ce *scope* est partagé entre tous les *flow*, donc toutes les feuilles.

Voici un flux en prenant comme exemple notre canal _zigbee2mqtt/TH_bureau_, avec en arrivé le débugueur (imprimé à droite)

![Le message en sorti du canal décomposé dans un objet javascript.](/img/2021/01/image.png)

*Le message en sorti du canal décomposé dans un objet javascript.*

En Output, l’option _parsed JSON object_ a été sélection afin d’obtenir un objet javascript comme on peut le voir dans le débugueur.

On retrouve nos 6 grandeurs: le niveau de batterie, l’humidité, la qualité de la connexion, la pression atmosphérique, la température et le niveau de tension de la batterie en mV.

L’objectif sera d’obtenir le même type de message pour tous les capteurs qui ont la même fonction. Ainsi un unique réseau de nœud **s** sera nécessaire pour l’ensemble de vos capteurs: moins de nœuds, donc moins de code, donc moins de bug et maintenance!

La qualité n’est pas toujours envoyé, donc il faudra la rajouter lorsqu’elle manque. Par ailleurs on peut rajouter un nom plus sympathique (comme _bureau_) au message. On va donc utiliser un nœud _switch_ pour détecter si _linkquality_ manque et des nœuds _change_ afin de rajouter les éléments manquants.

![l’objet _name_ a été rajouté, et linkquality rajouté si nécessaire](/img/2021/01/image-2.png)

*l’objet _name_ a été rajouté, et linkquality rajouté si nécessaire*

### …puis les insérer dans une table SQL

comme indiqué dans le précédent article, nous avons préalablement créé un schema `th` dans lequel les tables contenant les données de chaque capteur sera enregistré. L’objectif ici sera de créer ces tables si elles manquent, puis d’insérer les nouvelles données à chaque nouveau message.

Rajoutez un nœud postgresql sur votre flux. Après avoir double-cliqué, vous pouvez configurer Node-Red pour se connecter au serveur. Vous pouvez désactivé les outputs: on ne va faire qu’insérer de la donnée.

Entre le flux précédent (MQTT) et ce nœud postgresql, il faut faire deux nœuds:

- un nœud *switch* dans lequel on injectera les paramètres de la requête SQL et on les placera dans l’objet `msg.data`
- un nœud *template* dans lequel on assemblera notre requête et on la placera dans l’objet `msg.query`

Dans le premier nœud, vous créez un objet vide puis y placez vos données

![Un objet data vide est créé, puis on ajoute les données sous sensors](/img/2021/01/image-3.png)

*Un objet data vide est créé, puis on ajoute les données sous sensors*

Dans `msg.data.sensors` se trouve:

```
{"name":msg.name,
"battery": msg.payload.battery,
"humidity":msg.payload.humidity,
"pressure":msg.payload.pressure,
"temperature":msg.payload.temperature,
"voltage":msg.payload.voltage,
"linkquality":msg.payload.linkquality,
"msg_ts":$now()
}
```

Vous noterez que `msg_ts` a été rajouté afin de capturer le moment. On pourrait aussi utiliser `now` comme valeur par défaut dans postgresql, mais je préfère capturer le moment au plus proche de l’émission du message. Dans le second nœud, la propriété sélectionné sera `msg.query`, le format _Mustache template_, l’output sera du **texte** et la requête SQL ainsi:

```
CREATE TABLE IF NOT EXISTS th.{{{data.sensors.name}}}(
  ID         SERIAL    PRIMARY KEY    NOT NULL,
  battery    INT,
  humidity   REAL                     NOT NULL,
  pressure   REAL,
  temperature REAL                    NOT NULL,
  voltage    INT,
  linkquality INT,
  msg_ts     TIMESTAMP                NOT NULL
 );

INSERT INTO th.{{{data.sensors.name}}}(battery,
                                 humidity,
                                 pressure,
                                 temperature,
                                 voltage,
                                 linkquality,
                                 msg_ts)
VALUES ({{{data.sensors.battery}}}, 
        {{{data.sensors.humidity}}},
        {{{data.sensors.pressure}}},
        {{{data.sensors.temperature}}},
        {{{data.sensors.voltage}}},
        {{{data.sensors.linkquality}}},
        '{{{data.sensors.msg_ts}}}');

COMMIT;
```

La première partie de la requête est assez explicite: si la table nommé par le nom sous `msg.name` ou `data.sensor.name` n’existe pas, il faut la créer. On y retrouve nos 6 grandeurs ainsi que le temps et un identifiant unique. La seconde partie est la requête pour injecter la donnée. Le timestamp est entre des _single quotes_ puisque ce n’est pas un `int`, mais une chaîne de caractère. Il ne reste plus qu’à tout rassembler! On peut lier tous les canaux sur le même réseau, juste après avoir mis le nom.

![](/img/2021/01/image-4.png)

> **Voici un example à importer (Menu Sandwich → import)**

## Panel: voir les données historiques

On va recréer une application Panel comme dans le précédent article. Cette fois-ci le Dashboard sera beaucoup plus classique et facile à concevoir. Voici un Notebook Jupyter qui décrit point par point comment le créer.

Vous pouvez maintenant utiliser directement le code source de ce notebook comme la dernière fois en appelant:

```
panel serve yournotebook.ipynb --port 5006 --address localhost --show
```

cela devrait vous ouvrir une page dans votre navigateur avec uniquement le dernier élément appelé avec `.servable()`. Il serait judicieux de nettoyer le notebook avec les cellules de présentation si vous voulez accélérer le démarrage car chaque cellule est éxécutée, ou tout simplement d’aller voir [le notebook disponible sur GitHub](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/plotting_hist_data.ipynb).

## Intégrer

vous pouvez maintenant rajouter cette application au [service de votre Raspbrry Pi](/fr/2021/01/02/visualiser-les-mesures-de-capteurs-iot/#en-faire-un-service). En effet, bokeh, la bibliothèque sur laquelle est basée Panel, permet de lancer plusieurs applications en même temps. Elle génère automatiquement une page d’accueil qui permet à l’utilisateur de choisir entre celles-ci. On va donc modifier la ligne `ExecStart` du fichier `home_on_bokeh.service` et rajouter l’application (la modification ajoute le fichier `plotting_hist_data.py`):

```
sudo nano /etc/systemd/system/home_on_bokeh.service
```

```
Description=Serve panel to access zigbee sensor data

After=multi-user.target

 

[Service]

Type=simple

ExecStart=/home/pi/.local/bin/panel serve /home/pi/home_on_bokeh/plotting_live_data.py /home/pi/home_on_bokeh/plotting_hist_data.py --port 80 --address yourrpiaddress

Restart=on-abort

User=pi

 

[Install]

WantedBy=multi-user.target
```

Si vous utilisez des variables d’environnement pour l’authentification à la base de donnée, vous devez aussi les rajouter

```
sudo systemctl edit home_on_bokeh.service 
```

```
[Service]

Environment="MQTT_ADDRESS=localhost"

Environment="MQTT_USER=pi"

Environment="MQTT_PASSWORD=verycomplexpassphrase"

Environment="SQL_ADDRESS=localhost"

Environment="SQL_USER=pi"

Environment="SQL_PASSWORD=unguessablepassword"

Environment="BOKEH_ALLOW_WS_ORIGIN=raspberrypi.local:80,192.168.10.10:80"
```

Il ne reste plus qu’à redémarrer le service:

```
sudo systemctl restart home_on_bokeh.service
```

![](/img/2021/01/image-6.png)

Et voila! Cette seconde application est un peu plus lente au démarrage que la première, surtout que le pauvre RPi doit faire tourner node-red, postgresql et l’application en parallèle. Passer à un RPi 4 ne serait pas de trop. Mais si vous ne l’utilisez que pour la domotique, je conseille fortement de passer à **Home Assistant** et d’y rajouter les modules **Grafana** et **InfluxDB** pour arriver à un résultat similaire (quoi que, sans le fun 😉 ).
