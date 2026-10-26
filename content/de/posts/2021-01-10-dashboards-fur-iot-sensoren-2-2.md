---
title: Dashboards für IoT-Sensoren, Teil 2/2
author: hyamanieu
translationKey: iot-sensors-part-2
date: 2021-01-10T18:45:28+00:00
description: Wir zeichnen die auf dem Broker ankommenden Daten mit Node-Red und PostgreSQL auf und visualisieren sie anschließend mit den Python-Paketen von Holoviz.
slug: dashboards-fur-iot-sensoren-2-2
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
_Dieser Artikel ist die Fortsetzung von [Teil 1](/de/2021/01/02/dashboards-fur-iot-sensoren/)._

## Node-Red: Daten aufzeichnen

### MQTT-Nachrichten abholen…

Wie angekündigt, zeichnen wir die auf dem Broker ankommenden Daten mit [Node-Red](/de/2021/01/02/dashboards-fur-iot-sensoren/#node-red) in einer [PostgreSQL](/de/2021/01/02/dashboards-fur-iot-sensoren/#datenbank)-Datenbank auf.

Wenn die Einrichtung von Node-Red erfolgreich war, erreichen Sie es auf Port 1880 (z. B. `raspberrypi.local:1880`). Über die Benutzeroberfläche müssen Sie dann einrichten:

- die [PostgreSQL-Nodes von Digitaloak](https://flows.nodered.org/node/@digitaloak/node-red-contrib-digitaloak-postgresql)

Und das war's!

Ein MQTT-Node ist bereits vorhanden. Indem Sie einen `MQTT IN`-Node auf den aktuellen Flow-Tab ziehen, können Sie nach einem Doppelklick den _Broker_ konfigurieren, wie Sie es vorher in Python getan haben. Bearbeiten Sie dann den Node selbst mit dem korrekten Kanalnamen unter `Topic`, um auf diesen Sensor zu hören.

Node-Red arbeitet mit _Scopes_. Es gibt drei:

- **msg**: Dieser *Scope* verweist auf Objekte, die von allen miteinander verbundenen Nodes geteilt werden. Das Objekt, das uns hier interessiert, ist `msg.payload`. Es enthält die Nachricht aus Ihrem Quell-`MQTT IN`-Node.
- **flow**: Dieser *Scope* verweist auf alle Objekte, die Sie mit allen Nodes dieses Tabs teilen möchten, ob verbunden oder nicht.
- **global**: Wie der Name sagt, wird dieser *Scope* von allen *Flow*s geteilt, also allen Tabs.

Hier ist ein Flow mit dem Bürosensor als Beispiel, _zigbee2mqtt/TH_bureau_, dessen Ausgabe an den Debugger geht (rechts abgebildet):

![Die Ausgangsnachricht des MQTT-Kanals steckt in einem Javascript-Objekt.](/img/2021/01/image.png)

*Die Ausgangsnachricht des MQTT-Kanals steckt in einem Javascript-Objekt.*

Wählen Sie als Output _parsed JSON object_, um ein Javascript-Objekt wie im Debugger gezeigt zu erhalten.

Wir finden unsere 6 Größen: Batteriestand, Luftfeuchtigkeit, Verbindungsqualität, Luftdruck, Temperatur und die Batteriespannung in mV.

Unser Ziel ist hier, für jeden ähnlichen Sensor dasselbe Nachrichtenformat zu erhalten. Dann genügt ein einziger Flow für alle Sensoren: weniger Nodes bedeuten weniger Code, und damit weniger Bugs und weniger Wartung!

Die Verbindungsqualität wird von den Xiaomi-Sensoren nicht immer gesendet. Fehlt sie, müssen wir sie ergänzen. Wir können auch einen sprechenden Namen im `msg` hinzufügen, statt das lange `Topic` zu verwenden. Wir verwenden einen `switch`-Node, um zu erkennen, ob `linkquality` fehlt, und `change`-Nodes, um fehlende Elemente zu ergänzen.

![Das Objekt _name_ wurde hinzugefügt, ebenso linkquality, falls es fehlt](/img/2021/01/image-2.png)

*Das Objekt _name_ wurde hinzugefügt, ebenso linkquality, falls es fehlt*

### …und sie in SQL-Tabellen einfügen

Wie in der Einleitung des vorherigen Artikels beschrieben, haben wir ein Schema `th` erstellt, in dem wir pro Sensor eine Tabelle anlegen. Ziel unseres Flows ist es, fehlende Tabellen zu erstellen und dann die neuen Daten einzufügen.

Fügen Sie Ihrem Flow einen PostgreSQL-Node hinzu. Per Doppelklick geben Sie auch hier die Authentifizierungsparameter des Servers an. Sie können die Output-Funktion ebenfalls deaktivieren: Wir fügen nur Daten ein.

Zwischen dem `MQTT IN`-Node und dem `postgresql`-Node müssen Sie zwei Nodes hinzufügen:

- einen *switch*-Node, in dem wir die Parameter der SQL-Abfrage in das Objekt `msg.data` injizieren.
- einen *template*-Node, in dem wir unsere SQL-Abfrage schreiben und in das Objekt `msg.query` legen.

Im ersten Node erstellen Sie ein leeres Objekt und deklarieren dann Ihre Parameter.

![Ein leeres Objekt wird erstellt, dann fügen wir die Sensordaten hinzu](/img/2021/01/image-3.png)

*Ein leeres Objekt wird erstellt, dann fügen wir die Sensordaten hinzu*

Hinter `msg.data.sensors` steht:

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

Beachten Sie `msg_ts`. Es wurde hinzugefügt, um den Zeitpunkt zu erfassen. Wir hätten auch den Standardwert `now` von PostgreSQL verwenden können, aber ich ziehe es vor, eine Zeit so nah wie möglich am Sendevorgang der Nachricht zu erfassen.

Im zweiten Node wählen Sie die Eigenschaft `msg.query`, das Format _Mustache template_, als Ausgabeformat **Text** und schreiben die folgende SQL-Abfrage:

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

Der erste Teil der Abfrage ist selbsterklärend: Gibt es keine Tabelle mit dem Namen des Sensors, wird sie erstellt. Wir finden unsere 6 Größen, den Zeitpunkt und eine eindeutige ID.

Der zweite Teil der Abfrage ist das `insert`. Der Timestamp steht zwischen einfachen Anführungszeichen, da es sich um einen `string` handelt, nicht um einen `int`.

Jetzt müssen Sie nur noch alles zusammenfügen! Wir können mehrere Kanäle mit demselben `switch`-Node verbinden, nachdem wir den Sensornamen vergeben haben, wie im Bild unten zu sehen.

![](/img/2021/01/image-4.png)

> **Hier ist ein Beispiel zum Importieren (Sandwich-Menü → Import)**

## Panel: historische Daten betrachten

Wie bei den Live-Daten können wir eine Panel-Anwendung bauen, um die in der Datenbank gespeicherten Sensormessungen zu visualisieren. Diesmal wird das Dashboard klassischer und einfacher zu implementieren sein.

Unten ein Notebook, das Schritt für Schritt beschreibt, wie man es baut.

Wie vorher können Sie dieses Notebook als Quellcode Ihrer Anwendung verwenden:

```
panel serve yournotebook.ipynb --port 5006 --address localhost --show
```

Der obige Befehl öffnet eine Webseite, die nur das letzte Element anzeigt, das mit der Methode `.servable()` aufgerufen wurde. Es wäre klüger, das Notebook etwas zu bereinigen, da jede Zelle beim Aufruf des obigen Befehls ausgeführt wird. Oder Sie verwenden direkt das auf [GitHub](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/plotting_hist_data.ipynb) verfügbare Notebook.

## Einbinden

Sie können diese Anwendung jetzt [zum Dienst Ihres Raspberry Pi hinzufügen](/de/2021/01/02/dashboards-fur-iot-sensoren/#daraus-einen-dienst-machen). Bokeh, die Bibliothek hinter Panel, erlaubt es, mehrere Anwendungen parallel zu betreiben. Sie erzeugt automatisch eine Startseite, auf der Ihre Nutzer die Anwendung auswählen können.

Ändern Sie einfach die Zeile `ExecStart` in Ihrer Datei `home_on_bokeh.service`, indem Sie die Anwendung hinzufügen (die Änderung fügt `plotting_hist_data.py` hinzu):

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

Wenn Sie Umgebungsvariablen für die Authentifizierung an Ihrer Datenbank verwendet haben, müssen Sie sie ergänzen:

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

Starten Sie jetzt Ihren Dienst neu:

```
sudo systemctl restart home_on_bokeh.service
```

![](/img/2021/01/image-6.png)

Voilà! Diese zweite Anwendung startet langsamer als die erste. Obendrein muss der arme RPi einen Node-Red-Server und einen PostgreSQL-Server betreiben.

Ein Wechsel zu einem RPi 4 wird sicher helfen. Wenn Sie aber ausschließlich Hausautomation betreiben, empfehle ich dringend, **Home Assistant** zu verwenden und die Module **Grafana** und **InfluxDB** zu ergänzen, um ein ähnliches Ergebnis zu erreichen (nun ja, ohne den Spaß 😉 ).
