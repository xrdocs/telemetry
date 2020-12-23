---
published: false
date: '2020-12-22 15:23 +0100'
title: 'IOS-XR power consumption monitoring: an ephemeral telemetry stack use case'
author: fcuiller
excerpt: >-
  This post will describe how a docker-based ephemeral telemetry stack has been
  built to monitor Cisco IOS-XR device power consumption.
position: hidden
tags:
  - iosxr
---
### Introduction

Power efficiency is a big topic as it can literally [help customers save millions](https://blogs.cisco.com/sp/how-cisco-silicon-one-can-help-you-save-millions). Recently there has been several announcements about Cisco’s Silicon One and [how it compares to a previous generation of chip](https://blogs.cisco.com/sp/making-an-eco-friendly-network-with-cisco-silicon-one) on this specific point.  
This post will describe how a docker-based ephemeral telemetry stack has been built to monitor Cisco IOS-XR device power consumption.

### Context

As part of an upcoming CPOC (Customer Proof Of Concept) engagement, one of our customers wants to benchmark routers power consumption. Most of time, TCO (Total Cost of Ownership) calculations are made based on estimations. Cisco do provide such numbers through [Cisco Power Calculator tool](https://cpc.cloudapps.cisco.com/cpc/). However, they may vary based on several factors like ambient temperature, traffic load and pattern, optics, etc. For a fully loaded ASR 9906 with latest generation of RSP and linecards, there can be a 50% difference between the typical scenario (27C ambient temperature with 50% linerate IMIX traffic) and the worst-case scenario (50/55C)!  
For this multi-vendor assessment, apples to apples comparison is required. A power meter or a smart PDU could be installed, but they require additional hardware to be purchased and installed on the premises. Instead we decided to use Model Driven Telemetry to monitor in real time Cisco's device power consumption: it’s already available, free and we only need extra configuration. The last missing piece is a collector.

### Docker-compose telemetry stack

To receive and consume power data from our routers, I decided to build an ephemeral telemetry stack which can be brought on-demand on a server or a laptop. As CPOC will be delivered over a one-week period, there is no need to build a full-blown and permanent solution.  
I reused the same popular components: Telegraf, InfluxDB and Grafana. This time I decided to add Chronograf to quickly see the data stored in InfluxDB. I also leveraged Docker to build this stack and especially Docker Compose. As described in [Docker’s documentation](https://docs.docker.com/compose/):  
> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration.

There was some work done by [Jeff Kehres](https://github.com/jkehres) available on Github. I reused it to add Telegraf and I removed the docker volumes: as this stack is ephemeral, persistent storage is not required. Full code and documentation can be found [here](https://github.com/fcuiller/docker-compose-telegraf-influxdb-grafana). Here is the raw docker-compose.yaml file:  

`
version: "2"
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - '3000:3000'
    volumes:
      - ./grafana-provisioning/:/etc/grafana/provisioning
    depends_on:
      - influxdb
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
  influxdb:
    image: influxdb
    container_name: influxdb
    restart: always
    ports:
      - '8086:8086'
    environment:
      - INFLUXDB_DB=telemetry
      - INFLUXDB_USER=admin
      - INFLUXDB_PASSWORD=admin
      - INFLUXDB_ADMIN_ENABLED=true

      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
  chronograf:
    image: chronograf:latest
    container_name: chronograf
    ports:
      - '127.0.0.1:8888:8888'
    depends_on:
      - influxdb
    environment:
      - INFLUXDB_URL=http://influxdb:8086
      - INFLUXDB_USERNAME=admin
      - INFLUXDB_PASSWORD=admin
  telegraf:
    image: telegraf
    container_name: telegraf
    restart: always
    ports:
      - '57100:57100'
      - '57500:57500'
    environment:
      HOST_PROC: /rootfs/proc
      HOST_SYS: /rootfs/sys
      HOST_ETC: /rootfs/etc
    volumes:
     - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
     - /var/run/docker.sock:/var/run/docker.sock:ro
     - /sys:/rootfs/sys:ro
     - /proc:/rootfs/proc:ro
     - /etc:/rootfs/etc:ro
`

To bring up the stack, simply run:

`
docker-compose up -d
`

This will spawn 4 containers to create the ephemeral telemetry stack:

`
{21/12/20 11:58}fcuillers-MacBook-Pro:~/Dev/telemetry fcuiller% docker-compose up -d
Creating network "telemetry_default" with the default driver
Creating telegraf   ... done
Creating influxdb ... done
Creating chronograf ... done
Creating grafana    ... done
`

With current configuration, following ports are exposed:

**Host Port**|**Service**
:-----:|:-----:
3000|Grafana
8086|InfluxDB
57100, 57500|Telegraf
127.0.0.1:8888|Chronograf

Port 57100 is used for TCP transport while 57500 is used for gRPC dial-out. telegraf.conf file can be updated to fine tune configuration.   

Once stack is started, following containers should be up:

`
{21/12/20 15:27}fcuillers-MacBook-Pro:~/Dev/telemetry fcuiller% docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                                                                              NAMES
fa0fa0f61c54   grafana/grafana:latest   "/run.sh"                3 seconds ago   Up 2 seconds   0.0.0.0:3000->3000/tcp                                                             grafana
88f308900623   chronograf:latest        "/entrypoint.sh chro…"   3 seconds ago   Up 2 seconds   0.0.0.0:8888->8888/tcp                                                             chronograf
032555f2ac94   telegraf                 "/entrypoint.sh tele…"   4 seconds ago   Up 3 seconds   8092/udp, 0.0.0.0:57100->57100/tcp, 8125/udp, 8094/tcp, 0.0.0.0:57500->57500/tcp   telegraf
320d88dbfcf3   influxdb                 "/entrypoint.sh infl…"   4 seconds ago   Up 3 seconds   0.0.0.0:8086->8086/tcp                                                             influxdb
`


### IOS-XR models

### Router configuration

### I’ve got the power!

### Conclusion
This was another use case of telemetry, and we could imagine monitoring other KPI with Cisco IOS-XR environmental model like temperature, fan speed or voltages. Docker compose is a very flexible and agile solution when it comes to bring up an application stack. Combining both allows to quickly gain useful information and insights from the infrastructure.
