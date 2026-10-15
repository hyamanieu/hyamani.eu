---
title: Dashboards for IoT sensors
author: hyamanieu
translationKey: iot-sensors-part-1
date: 2021-01-02T15:53:30+00:00
description: "How to record the measurements of sensors that send their data asynchronously, visualize their live status or analyze their past measurements? We'll do that with Node-Red, MQTT, and the python holoviz libraries."
slug: dashboards-for-iot-sensors
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
_This is part 1. [Part 2 is there](/2021/01/10/dashboards-for-iot-sensors-part-2-2/)._

## Introduction

It’s been two years since I’ve written an article. It’s never too late to start again, especially for a subject that covers the whole chain from physical measurement to its analysis in a simple way.

We will therefore talk about a subject that seems particularly important to me: how to record the measurements of sensors that send their data asynchronously, visualize their live status or analyze their past measurements.

To do this, we will talk about humidity, temperature and atmospheric pressure sensors in the field of home automation. The objective will not be to reinvent Home Automation (HA), but to explore **Node-Red** and the excelent holoviz **Holoviews** & **Panel** libraries using sensors costing a few euros. The same principle could be applied to any field.

In a few sections:

1. Hardware: from *Zigbee* to *MQTT*
2. Dashboard live data using *Holoviews stream*
3. Record data using *Node-Red and* *PostgreSQL*
4. Dashboard historical data

Section 3 and 4 are for my next blog post.

## Hardware

- a computer that can run [Zigbee2MQTT[EN]](https://www.zigbee2mqtt.io/getting_started/running_zigbee2mqtt.html), here a **RaspberryPi 3B+**.
- A zigbee sniffer. I opted for one advised on a guide by[ projetsdiy.fr: le CC2531 de Texas Instrument[FR]](https://projetsdiy.fr/test-pont-zigbee2mqtt-hack-gateway-xiaomi-smart-home-aqara-assemblage-systeme-alarme-node-red/) (links are certainly sponsored), [but others are compatible [EN].](https://www.zigbee2mqtt.io/information/supported_adapters.html)
- A set of sensors using the Zigbee protocol and preferably [compatible with the Zigbee2MQTT library [EN]](https://www.zigbee2mqtt.io/information/supported_devices.html). Here, a set of temperature, humidity, pressure sensors by Xiaomi are used.

The raspberryPi 3B+ will not only be a **Zigbee** coordinator and the **MQTT _broker_**, but also serve **Node-Red**, **PostgreSQL** and our self-developped **Panel applications**. Within your local network, you can of course distribute these services on different hosts.

## Setting up

We won’t be talking details here. Each module is thoroughly documented by each project website. I’ll be giving only an overview with specifics for this project.

### Zigbee sniffer and MQTT broker

The documented is very good on the main website: https://www.zigbee2mqtt.io.

A small tip: note the time and the order for which you reset your sensors. You can then find their id back on the journal using `journalctl -u zigbee2mqtt.service`. Then you can give them friendly names in the configuration file `/opt/zigbee2mqtt/data/configuration.yaml`.

You must also add the `retain` option for each sensor. We need your broker to record the last sent data by your sensor. Otherwise, if nobody is listening, the information is lost for ever. In the case of Zigbee sensor, new data may take up to one hour to be sent (remember, it’s low consumption). When rebooting your application, you may still want to know what was the last data and when it was.

> Note: Xiaomi sensors have a small button to send current measurements right away.

Extract from the configuration file:

```
devices:
  '0x00158d00054692fd': 
    friendly_name: TH_Chambre
    retain: true
```

_TH_ for _**T**emperature, **H**umidit_y.

### Database

I have opted for PostgreSQL that you can simply install with `sudo apt install postgresql postgresql-contrib`

I advise you to make a user, let’s call it _pi_, with remote access to a database called _sensors_. I have added a schematics _th_ where one table per sensor will be automatically made. We’ll come back to in in the section about **Node-Red**.

### Node-Red

The documentation is here as well thorough: https://nodered.org/docs/getting-started/raspberrypi

### Python 3.7 and its libraries

Luckily, the last Raspbian version is compatible with Python 3.7. Simply calling `sudo apt install python3` will suffice (python2 stays the default engine).

Then:

- `pip3 install paho-mqtt` for MQTT.
- `pip3 install holoviews panel hvplot` for the dashboard
- `pip3 install sqlalchemy psycopg2` to access the database.

On your own machine, I strongly invite you to use [conda][1]‘s virtual environment to install these. Also you can use Jupyter Notebook. One of the nice features of Panel is to be editable live, and you can use the notebook as the source code for your application.

## Visualize live data

### A little bit of python…

You can reproduce herebelow notebook or check it [on github here](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/live_blog_en.ipynb). To executed from a computer without jupyter (like a RPi), you can convert it to a python file with `jupyter nbconvert yournotebook.ipynb --to python`

### A full fledge Dashboard

To make it a Panel application, you must call the `servable` method from a Panel layout as the last line of your jupyter notebook. Using above example:

```
layout=pn.panel(room_plots)
layout.servable()
```

Then from your RPi terminal (or any other server ), call:

```
panel serve yournotebook.ipynb --port 8080 --address youraddress
```

If you want to make it accessible from your local network like a normal http website (on port 80), you must bind the python3.7 execute as follows:

```
sudo setcap 'cap_net_bind_service=+ep' /usr/bin/python3.7
```

> Note that **all** python3.7 application will have access to ports under 1024, which can be considered as a security issue.

You can find [my whole application on GitHub following this link.](https://github.com/hyamanieu/examples_sensor_data_to_Panel_dashboard/blob/main/plotting_live_data.ipynb) You’ll need to adapt it to your configuration of course!

### Make it a service

You now have a new app. But you want it to start automatically without having to prompt passwords and such. Then make it a service!

It is very simple. You first need to create a file in the right folder, as follows:

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

change `yourrpiaddress` and, if necessary, `--port 80` for an accessible port. You now need to define the broker address and authentification as well as Bokeh paramters as environmental variables:

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

This file is accessible under `/etc/systemd/system/home_on_bokeh.service.d` .

> Note the `BOKEH_ALLOW_WS_ORIGIN` variable: without it, your thing isn’t accessible! Change the address here by your real address.

If you’d prefer having nobody reading its content (and passphrases!), changing access rights is not enough, see this [*server fault* response](https://serverfault.com/a/910655). You can now start the service:

```
>> sudo systemctl start home_on_bokeh
```

enable it for automated start:

```
>> sudo systemctl enable home_on_bokeh
```

check that everything goes well (no `sudo` is necessary)

```
>> systemctl status home_on_bokeh
```

And check the logs:

```
>> journalctl -u home_on_bokeh -r
```

You can now access the dashboard from any of your devices present on your local network!

Next time, I’ll discuss how Node-Red can be used to record the data then make a new dashboard to look at them (hint: it’s already on GitHub).

[1]: https://docs.conda.io/en/latest/miniconda.html
