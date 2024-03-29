---
title: "Logging and visualization of distributed sensor measurements"
layout: post
summary: Environmental sensor control with a RasbperryPi
tags:
  - hardware
  - raspberrypi
date: 2017-08-20
---


With the availability of more powerful but low power hardware like the raspberry pi and generic data store and visualization tools, a room climate data logger can be easily implemented. We use the following compontents:

  * measurement and data transmission (RaspberryPI, hardware sensors, Python script)
  * timeseries data storage (InfluxDB)
  * visualization (Grafana)

## Data collection

The module [klimalogger](https://github.com/wuan/klimalogger) was created to have a configurable data collection mechanism on the nodes with the following features:

  * support various sensors
  * easily expandable to support other sensors
  * fully configurable
  * buffers data if timeseries data storage is unavailable

## Supported sensors

### Temperature/Humidity

We used a high precision SHT1x sensor for temperature and humidity measurements. As a driver we used the [rpiSht1x](https://bitbucket.org/lunobili/rpisht1x) module by Luca Nobili.

### Atmospheric pressure

The atmospheric pressure was detected by a BMP085 sensor. The [Adafruit BMP085 library](https://github.com/adafruit/Adafruit-BMP085-Library) was used to read out data.

## Setup of timeseries data store

We chose InfluxDb as our timeseries data store, which is available [here](https://www.influxdata.com/downloads/) for download for a variety of systems and architectures. Just download and install the software.

Now we create a database for storage of our measurements:

```
$ influx
> CREATE DATABASE klima
> CREATE USER klimalogger WITH PASSWORD '<password>'
> GRANT WRITE ON klima TO klimalogger
```

## Setup measurement(s)

Go to a client and install the software to take measuerments

First we install the SHT1x driver module from sources:

```sh
git clone https://bitbucket.org/lunobili/rpisht1x.git
cd rpisht1x/src
python3 setup.py sdist
easy_install-3.x dist/rpiSht1x-1.3.tar.gz 
```

The BMP085 driver module can be installed directly from PyPi:

```sh
pip3 install Adafruit_BMP
```

The I2C kernel module should be enabled in the Advanced section of `raspi-config`.

The we install our klimalogger module and create the directory for the local cache:

```sh
pip3 install klimalogger
mkdir -p /var/cache/klimalogger
```

 To configure the modules for our purpose, we create a configuration file `/etc/klimalogger.conf` with the following content:


```
[client]
location_name=living room
sensors=sht1x, bmp085

[store]
host=datastore.home
port=8086
name=klima
username=klimalogger
password=<password>

[log]
path=/var/cache/klimalogger

[sht1x_sensor]
data_pin=18
sck_pin=16

[bmp085_sensor]
elevation=535
```

You can check now if the local part is working by entering

```
klimalogger --check
```

To have a periodical measurement, create a file /etc/cron.d/klimalogger with the following content:

```
* * * * * root /usr/local/bin/klimalogger >/dev/null 2>/dev/null
```

Now a measurement should be taken every minute.

## Check that measurement data arrives at the data store

You should see a new measurement name data in the data store.

```
$ influx
> USE klima
> SHOW MEASUREMENTS
name: measurements
------------------
name
data
```

Then you can look at the incoming data

```
> SELECT * FROM data

name: data
----------
time                 calculated  host    location     sensor  type               unit  value
1473496742252090880  False       hermes  living room  SHT1x   relative humidity  %     63.35
1473496742252090880  True        hermes  living root  SHT1x   dew point          °C    16.64
1473496742252090880  False       hermes  living room  SHT1x   temperature        °C    24.03
1473496743777595904  True        hermes  living room  SHT1x   dew point          °C    16.65
1473496743777595904  False       hermes  living room  SHT1s   temperature        °C    24.06
1473496743777595904  False       hermes  living roo   SHT1x   relative humidity   %    63.3
```

The Python project used to acquire and store the data is freely available:

https://github.com/wuan/klimalogger
