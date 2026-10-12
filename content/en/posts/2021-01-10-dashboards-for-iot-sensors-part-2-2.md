---
title: Dashboards for IoT sensors, part 2/2
author: hyamanieu
type: post
date: 2021-01-10T18:45:28+00:00
excerpt: We record data coming on the broker by using Node-Red and PostgreSQL., then visualize them with the holoviz python packages.
url: /2021/01/10/dashboards-for-iot-sensors-part-2-2/
featured_image: /img/2021/01/image.png
categories:
  - Panel
tags:
  - holoviews
  - mqtt
  - node-red
  - panel
  - postgresql

---
_This article is a follow-up of [part 1](/2021/01/02/dashboards-for-iot-sensors/)._

## Node-Red: record data

### Fetch MQTT messages…

As introduced, we’re going to record data coming up on the broker by using [Node-Red](/2021/01/02/dashboards-for-iot-sensors/#node-red) on a [PostgreSQL](/2021/01/02/dashboards-for-iot-sensors/#database) database.

If setting-up Node-Red is successfull, you can reach it on port 1880 (e.g. `raspberrypi.local:1880`). Then over its user interface, you need to set-up:

- postgresql [nodes by Digitaloak](https://flows.nodered.org/node/@digitaloak/node-red-contrib-digitaloak-postgresql)

And that’s it!

An MQTT node already exists. By placing a `MQTT IN` node on the current flow tab, you can configure the _broker_ after double-clicking on it as you did previously in python. Then edit the node itself with the correct channel name under `Topic` to listen to this sensor.

Node Red works with _scopes_. There are three:

- **msg**: this *scope* points to objects shared among all nodes that are linked with each other. The object that interests us here is `msg.payload`, it holds the message coming from your source `MQTT IN` node.
- **flow**: this *scope* points to all objects you wish to share with all nodes on that tab, whether they’re linked or not.
- **global**: as its name indicates it, this *scope* is shared with all *flow*s, thence all tabs.

Here is a flow using the office sensor as example, _zigbee2mqtt/TH_bureau_, which outputs to the debugger (printed on the right):

![The output message from the MQTT canal is held in a javascript object.](/img/2021/01/image.png)

*The output message from the MQTT canal is held in a javascript object.*

As Output, use _parsed JSON object_ to receive a javascript object as shown on the debugger.

We find our 6 features: battery level, humidity, link quality, air pressure, temperature and the battery voltage in mV.

Our objective here is to get the same message format for each similar sensor. We can then use a unique flow for all sensors: fewer nodes means less code, so also fewer bugs and less maintenance!

Link quality is not always sent by the Xiaomi sensors. So we need to add it should it be missing. We can also add a friendly name in the `msg` rather than using the long `Topic`. We use a `switch` node to detect whether `linkquality` is lacking and `change` nodes to add missing elements.

![The _name_ object is added, linkquality as well if missing](/img/2021/01/image-2.png)

*The _name_ object is added, linkquality as well if missing*

### …and insert them in SQL tables

As indicated in the introduction of the preceding article, we have created a `th` schema within which we will add a table per sensor. Our objective in our flow is to create the tables if they’re missing, then insert the new data.

Add a PostgreSQL node to your flow. By double clicking, you can here as well indicate the server authentication parameters. You can also deactivate the Output function: we are only inserting.

Between the `MQTT IN` node and the `postgresql` node, you need to add two nodes:

- a *switch* node within which we inject the SQL query parameters inside the `msg.data` object.
- a *template* node within which we write our SQL query and put it in the `msg.query` object.

In the first node, you create an empty object then declare your parameters.

![An empty object is created, then we add sensor data within it](/img/2021/01/image-3.png)

*An empty object is created, then we add sensor data within it*

Behind `msg.data.sensors` is:

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

Note `msg_ts`. It has been added to capture the datetime. We could have also used the default `now` value of PostgreSQL, but I prefer capturing a time as close as possible to the emitting event.

In the second node, select the `msg.query` property, use the _Mustache template_ format, select **text** as output format and write the following SQL query:

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

The first part of the query is explicit: if there is no table named after the sensor, it should be created. We find our 6 features, the datetime and a unique ID.

The second part of the request is the `insert`. Timestamp is between single quotes as it is a `string`, not an `int`.

Now all you need to do is to put everything together! We can link several channels to the same `switch` node after giving the sensor names as can be seen in the image herebelow.

![](/img/2021/01/image-4.png)

> **Here is an example to import (Sandwich menu → import)**

## Panel: watch historical data

As we did for live data, we can build a Panel application to visualize sensor measurements held in the database. This time the Dashboard will be more classic and easier to implement.

Herebelow a notebook describing step after step how to build it.

As previously, you can use this notebook as code source for your application:

```
panel serve yournotebook.ipynb --port 5006 --address localhost --show
```

above command will open a webpage showing only the last element, called with the `.servable()` method. It would be smarter to clean the notebook a little as every cell will be executed upon calling above command. Or you could directly use the notebook available on [GitHub](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/plotting_hist_data.ipynb).

## Implement

You can now add this application [to your raspberry pi service](/2021/01/02/dashboards-for-iot-sensors/#make-it-a-service). Bokeh, the library behind Panel, let you serve several applications in parallel. It automatically generates a home webpage letting your users pick the application.

Simply modify the `ExecStart` line from your `home_on_bokeh.service` file by adding the application (the modification adds `plotting_hist_data.py`):

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

If you’ve used environment variables for authenticating to your database, you must add them:

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

Now restart your service:

```
sudo systemctl restart home_on_bokeh.service
```

![](/img/2021/01/image-6.png)

There you go! This second application is slower to start than the first one. On top of it, the poor RPi must run a Node-Red server and a postgresql server.

Swaping to a RPi 4 will certainly help. But if all you do concerns domotic applications, I strongly suggest you to use **Home Assistant** and add the **Grafana** and **InfluxDB** modules to reach a similar result (well, without all the fun 😉 ).
