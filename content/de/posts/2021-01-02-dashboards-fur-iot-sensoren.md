---
title: Dashboards für IoT-Sensoren
author: hyamanieu
translationKey: iot-sensors-part-1
date: 2021-01-02T15:53:30+00:00
description: "Wie zeichnet man Messwerte von Sensoren auf, die ihre Daten asynchron senden, zeigt ihren Live-Status oder analysiert ihre früheren Messungen? Wir machen das mit Node-Red, MQTT und den Python-Bibliotheken Holoviz."
slug: dashboards-fur-iot-sensoren
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
series: ['iot-sensors']
---
_Dies ist Teil 1. [Teil 2 finden Sie hier](/de/2021/01/10/dashboards-fur-iot-sensoren-2-2/)._

## Einführung

Es ist zwei Jahre her, dass ich einen Artikel geschrieben habe. Es ist nie zu spät, wieder anzufangen. Besonders für ein Thema, das die gesamte Kette von der physikalischen Messung bis zu ihrer Analyse auf einfache Weise abdeckt.

Wir sprechen also über ein Thema, das mir besonders wichtig erscheint: wie man die Messwerte von Sensoren aufzeichnet, die ihre Daten asynchron senden, ihren Live-Status visualisiert oder ihre früheren Messungen analysiert.

Dazu sprechen wir über Feuchtigkeits-, Temperatur- und Luftdrucksensoren im Bereich der Hausautomation. Ziel ist es nicht, Home Automation (HA) neu zu erfinden, sondern **Node-Red** und die hervorragenden Holoviz-Bibliotheken **Holoviews** & **Panel** mit Sensoren für ein paar Euro zu erkunden. Dasselbe Prinzip lässt sich auf jedes beliebige Gebiet anwenden.

In wenigen Abschnitten:

1. Hardware: von *Zigbee* zu *MQTT*
2. Live-Daten im Dashboard mit *Holoviews stream*
3. Daten aufzeichnen mit *Node-Red* und *PostgreSQL*
4. Dashboard mit historischen Daten

Die Abschnitte 3 und 4 folgen in meinem nächsten Blogartikel.

## Hardware

- ein Rechner, auf dem [Zigbee2MQTT (EN)](https://www.zigbee2mqtt.io/getting_started/running_zigbee2mqtt.html) laufen kann, hier ein **RaspberryPi 3B+**.
- einen Zigbee-Sniffer. Ich habe mich für einen entschieden, der in einer Anleitung von [projetsdiy.fr empfohlen wird: den CC2531 von Texas Instrument (FR)](https://projetsdiy.fr/test-pont-zigbee2mqtt-hack-gateway-xiaomi-smart-home-aqara-assemblage-systeme-alarme-node-red/) (Links sind vermutlich gesponsert), [aber andere sind kompatibel (EN)](https://www.zigbee2mqtt.io/information/supported_adapters.html).
- einen Satz Sensoren, die das Zigbee-Protokoll nutzen und vorzugsweise [mit der Zigbee2MQTT-Bibliothek kompatibel sind (EN)](https://www.zigbee2mqtt.io/information/supported_devices.html). Hier werden Temperatur-, Feuchtigkeits- und Drucksensoren von Xiaomi verwendet.

Der RaspberryPi 3B+ dient nicht nur als **Zigbee**-Koordinator und **MQTT-_Broker_**, sondern hostet auch **Node-Red**, **PostgreSQL** und unsere selbst entwickelten **Panel-Anwendungen**. Im lokalen Netzwerk können Sie diese Dienste natürlich auf verschiedene Rechner verteilen.

## Einrichtung

Hier gehen wir nicht ins Detail. Jedes Modul ist auf der jeweiligen Projekt-Website ausführlich dokumentiert. Ich gebe nur einen Überblick mit den Besonderheiten dieses Projekts.

### Zigbee-Sniffer und MQTT-Broker

Die Dokumentation auf der Haupt-Website ist sehr gut: https://www.zigbee2mqtt.io.

Ein kleiner Tipp: Notieren Sie sich Uhrzeit und Reihenfolge, in der Sie Ihre Sensoren zurücksetzen. Anschließend finden Sie ihre IDs im Journal mit `journalctl -u zigbee2mqtt.service` wieder. Dann können Sie ihnen in der Konfigurationsdatei `/opt/zigbee2mqtt/data/configuration.yaml` sprechende Namen geben.

Sie müssen außerdem die Option `retain` für jeden Sensor hinzufügen: Der Broker soll die zuletzt gesendeten Daten des Sensors speichern. Andernfalls geht die Information für immer verloren, wenn gerade niemand zuhört. Bei Zigbee-Sensoren kann das Senden neuer Daten bis zu einer Stunde dauern (denken Sie daran: geringer Verbrauch). Beim Neustart Ihrer Anwendung möchten Sie womöglich dennoch wissen, was die letzten Daten waren und wann sie gesendet wurden.

> Hinweis: Xiaomi-Sensoren haben einen kleinen Knopf, um die aktuellen Messwerte sofort zu senden.

Auszug aus der Konfigurationsdatei:

```
devices:
  '0x00158d00054692fd': 
    friendly_name: TH_Chambre
    retain: true
```

_TH_ steht für _**T**emperatur, **H**umidity (Feuchte)_.

### Datenbank

Ich habe mich für PostgreSQL entschieden, das sich einfach mit `sudo apt install postgresql postgresql-contrib` installieren lässt.

Ich empfehle, einen Benutzer, nennen wir ihn _pi_, mit Fernzugriff auf eine Datenbank namens _sensors_ anzulegen. Ich habe ein Schema _th_ hinzugefügt, in dem automatisch eine Tabelle pro Sensor angelegt wird. Darauf kommen wir im Abschnitt zu **Node-Red** zurück.

### Node-Red

Die Dokumentation ist ebenfalls gründlich: https://nodered.org/docs/getting-started/raspberrypi

### Python 3.7 und seine Bibliotheken

Glücklicherweise ist die neueste Raspbian-Version mit Python 3.7 kompatibel. Ein einfaches `sudo apt install python3` genügt (python2 bleibt die Standard-Engine).

Dann:

- `pip3 install paho-mqtt` für MQTT.
- `pip3 install holoviews panel hvplot` für das Dashboard
- `pip3 install sqlalchemy psycopg2` für den Zugriff auf die Datenbank.

Auf Ihrem eigenen Rechner empfehle ich dringend, für die Installation die virtuelle Umgebung von [conda][1] zu verwenden. Sie können auch Jupyter Notebook nutzen. Eine der netten Funktionen von Panel ist die Live-Bearbeitbarkeit: Sie können das Notebook als Quellcode Ihrer Anwendung verwenden.

## Live-Daten visualisieren

### Ein wenig Python…

Sie können das untenstehende Notebook nachvollziehen oder [hier auf GitHub](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/live_blog_en.ipynb) einsehen. Um es auf einem Rechner ohne Jupyter (wie einem RPi) auszuführen, können Sie es mit `jupyter nbconvert yournotebook.ipynb --to python` in eine Python-Datei umwandeln.

### Ein vollwertiges Dashboard

Um daraus eine Panel-Anwendung zu machen, müssen Sie als letzte Zeile Ihres Jupyter-Notebooks die Methode `servable` auf einem Panel-Layout aufrufen. Mit dem obigen Beispiel:

```
layout=pn.panel(room_plots)
layout.servable()
```

Dann rufen Sie im Terminal Ihres RPi (oder eines anderen Servers) auf:

```
panel serve yournotebook.ipynb --port 8080 --address youraddress
```

Wenn Sie es wie eine normale HTTP-Website im lokalen Netzwerk (auf Port 80) erreichbar machen wollen, müssen Sie die Python-3.7-Binärdatei wie folgt binden:

```
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.7
```

> Beachten Sie, dass **alle** Python-3.7-Anwendungen Zugriff auf Ports unter 1024 erhalten, was als Sicherheitsproblem gewertet werden kann.

Meine [vollständige Anwendung finden Sie auf GitHub unter diesem Link.](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/plotting_live_data.ipynb) Sie müssen sie natürlich an Ihre Konfiguration anpassen!

### Daraus einen Dienst machen

Sie haben nun eine neue App. Aber Sie möchten, dass sie automatisch startet, ohne dass Passwörter und dergleichen abgefragt werden müssen. Machen Sie daraus also einen Dienst!

Das ist ganz einfach. Zuerst müssen Sie eine Datei im richtigen Ordner anlegen, wie folgt:

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

Ändern Sie `yourrpiaddress` und, falls nötig, `--port 80` in einen erreichbaren Port. Nun müssen Sie noch die Broker-Adresse und Authentifizierung sowie die Bokeh-Parameter als Umgebungsvariablen definieren:

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

Diese Datei ist unter `/etc/systemd/system/home_on_bokeh.service.d` erreichbar.

> Beachten Sie die Variable `BOKEH_ALLOW_WS_ORIGIN`: Ohne sie ist Ihre Anwendung nicht erreichbar! Ersetzen Sie die Adresse hier durch Ihre echte Adresse.

Wenn Sie verhindern möchten, dass jemand den Inhalt (mit den Passwörtern!) liest, reichen geänderte Zugriffsrechte nicht aus (siehe diese [*server fault*-Antwort](https://serverfault.com/a/910655)). Jetzt können Sie den Dienst starten:

```
>> sudo systemctl start home_on_bokeh
```

und für den automatischen Start aktivieren:

```
>> sudo systemctl enable home_on_bokeh
```

Prüfen Sie, dass alles gut läuft (kein `sudo` nötig):

```
>> systemctl status home_on_bokeh
```

Und werfen Sie einen Blick in die Logs:

```
>> journalctl -u home_on_bokeh -r
```

Sie können nun von jedem Gerät in Ihrem lokalen Netzwerk auf das Dashboard zugreifen!

Das nächste Mal erkläre ich, wie Node-Red zum Aufzeichnen der Daten genutzt wird und wie daraus ein neues Dashboard entsteht (Hinweis: es ist bereits auf GitHub).

[1]: https://docs.conda.io/en/latest/miniconda.html
