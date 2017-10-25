---
published: true
date: '2017-10-25 10:12 -0700'
title: NCS1002 Telemetry deep dive
author: Viktor Osipchuk
excerpt: NCS1002 Telemetry deep dive
tags:
  - iosxr
  - NCS1002
  - Rosco
  - Telemetry
  - Visualization
  - monitoring
---
This tutorial continues the series of documents about [automation of configuration of Cisco optical products](https://xrdocs.github.io/programmability/tutorials/). The purpose of this document is to give you a lot of details about Telemetry on [NCS1002 (terminal device)](https://www.cisco.com/c/en/us/products/collateral/optical-networking/network-convergence-system-1000-series/datasheet-c78-733699.html). The goal is not only to give you some information about valuable sensor paths for NCS1002, but also to provide all the pieces for you to start exploring Telemetry on NCS1002 right away.

Model-driven Telemetry (MDT) provides a mechanism to stream data from an MDT-capable device to a destination(s). There are several core components of Streaming Telemetry you should know and understand:
- "Sensor path". Describes the data you want your NCS1002 to stream to a collector (for example, OSNR values from your line ports)
- "Transport protocol". Describes the protocol you want to use to deliver to your collector/controller the information you selected with sensor-paths (for example, TCP)
- "Encoder". Describes the format of the data on the wire (for example, GPB)
- "Initialization of streaming session". Describes who initiates the streaming of data from the router towards the collector. Two possible options are: dial-in mode (the collector initiates a session to the router and subscribes to data to be streamed out) or dial-out mode (the router initiates a session to the destinations based on the subscription.)
- "Subscription". Binds everything together for the router to start streaming data on the configured intervals.

There is no need to go in details about telemetry, as there are many different technical documents available in [xrdocs-telemetry](https://xrdocs.github.io/telemetry/), and you can also find configuration information on [cisco.com](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/telemetry/b-telemetry-cg-ncs5500-62x/b-telemetry-cg-ncs5500-62x_chapter_010.html). 

## Sensor Paths for NCS1002

With this generic understanding of telemetry, let’s now define sensor paths that could be valuable exactly for your optical deployments based on NCS1002. These paths provide valuable information about NCS1002 and that list should be pretty complete for you to start playing around and testing your optical DCI segments. Feel free to add or remove paths from the suggested list as well as optimize sample-interval timers according to your requirements.
Here is a list of sensor paths for NCS1002 with some basic explanation and print screens from Grafana for better visibility:

- <b>Monitoring of CPU utilization:</b> *Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization.*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/CPU_health.png)

With this model you’re able to get information about NCS1002 CPU load for 1-, 5- and 15-min intervals. 

- <b>Monitoring of memory usage:</b> *Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary* 

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Memory.png)

This model helps you to have information about NCS1002 free memory availability.

- <b>Alarms:</b> *Cisco-IOS-XR-alarmgr-server-oper:alarms/brief/brief-card/brief-locations/brief-location/active.* 

This path gives you information about currently active alarms in NCS1002. 

Here is an example of text-based output for active alarms on an NCS1002 under testing. You can modify your existing tools (or develop new) to collect this information either directly from a [TSDB (time-series database)](https://en.wikipedia.org/wiki/Time_series_database) like [InfluxDB](https://www.influxdata.com/) or from [Kafka platform](https://kafka.apache.org/), depending on your design. 

```
Summary: GPB(common) Message [172.16.1.1:27939(rosco_1)/Cisco-IOS-XR-alarmgr-server-oper:alarms/brief/brief-card/brief-locations/brief-location/active msg len: 3150]
{
    "Source": "172.16.1.1:27939",
    "Telemetry": {
        "node_id_str": "rosco_1",
        "subscription_id_str": "optical",
        "encoding_path": "Cisco-IOS-XR-alarmgr-server-oper:alarms/brief/brief-card/brief-locations/brief-location/active",
        "collection_id": 24180,
        "collection_start_time": 1508635312850,
        "msg_timestamp": 1508635312850,
        "collection_end_time": 1508635312858
    },
    "Rows": [
        {
            "Timestamp": 1508635312856,
            "Keys": {
                "node-id": "0/RP0/CPU0"
            },
            "Content": {
                "alarm-info_PIPELINE_EDIT": [
                    {
                        "clear-time": "-",
                        "clear-timestamp": 0,
                        "description": "Optics0/0/0/1 - Improper Removal",
                        "group": "controller",
                        "location": "0/0",
                        "set-time": "09/12/2017 23:30:07 PDT",
                        "set-timestamp": 1505284207,
                        "severity": "critical"
                    },
       {
                        "clear-time": "-",
                        "clear-timestamp": 0,
                        "description": "One Or More FPDs Need Upgrade Or Not In Current State",
                        "group": "fpd-infra",
                        "location": "0/0",
                        "set-time": "09/14/2017 01:28:05 PDT",
                        "set-timestamp": 1505377685,
                        "severity": "major"
                    },
                    {
                        "clear-time": "-",
                        "clear-timestamp": 0,
                        "description": "HundredGigECtrlr0/0/0/0 - Carrier Loss On The LAN",
                        "group": "ethernet",
                        "location": "0/0",
                        "set-time": "09/22/2017 07:46:09 PDT",
                        "set-timestamp": 1506091569,
                        "severity": "major"
                    },
       {
                        "clear-time": "-",
                        "clear-timestamp": 0,
                        "description": "Optics0/0/0/20 - Optics High Laser Bias",
                        "group": "controller",
                        "location": "0/0",
                        "set-time": "10/19/2017 10:46:23 PDT",
                        "set-timestamp": 1508435183,
                        "severity": "minor"
                    }
                ]
            }
        }
    ]
}
```

- <b>Chassis info:</b> *Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/attributes/basic-info.*

This path is useful to get information about the serial number and software version on NCS1002. You don’t need to stream this data very often.  

Here is an example output of this path for the NCS1002 under testing: 

```
Summary: GPB(common) Message [172.16.1.1:27939(rosco_1)/Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/attributes/basic-info msg len: 471]
{
    "Source": "172.16.1.1:27939",
    "Telemetry": {
        "node_id_str": "rosco_1",
        "subscription_id_str": "optical",
        "encoding_path": "Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/attributes/basic-info",
        "collection_id": 24175,
        "collection_start_time": 1508635312221,
        "msg_timestamp": 1508635312221,
        "collection_end_time": 1508635312225
    },
    "Rows": [
        {
            "Timestamp": 1508635312223,
            "Keys": {
                "name": "0"
            },
            "Content": {
                "description": "Network Convergence System 1002 20 QSFP28/QSFP+ slots",
                "firmware-revision": "",
                "hardware-revision": "V01",
                "is-field-replaceable-unit": "true",
                "model-name": "NCS1002-K9",
                "name": "Rack 0",
                "serial-number": "CAT1111A1AA",
                "software-revision": "6.2.2\n",
                "vendor-type": "1.3.6.1.4.1.9.12.3.1.3.1786"
            }
        }
    ]
}
```

- <b>Pluggables info:</b> *Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/slots/slot/cards/card/port-slots/port-slot/portses/ports/hw-components/hw-component/attributes/basic-info.*

This path is useful to get information about the serial number and hardware revision details about pluggables inserted into the platform. You don’t need to stream this data very often, but it might be valuable to collect information about installed pluggables on each platform for inventory. 

Here is a partial output of this path for the NCS1002 under testing: 

```
Summary: GPB(common) Message [172.16.1.1:41461(rosco_1)/Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/slots/slot/cards/card/port-slots/port-slot/portses/ports/hw-components/hw-component/attributes/basic-info msg len: 4611]
{
    "Source": "172.16.1.1:41461",
    "Telemetry": {
        "node_id_str": "rosco_1",
        "subscription_id_str": "100",
        "encoding_path": "Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/slots/slot/cards/card/port-slots/port-slot/portses/ports/hw-components/hw-component/attributes/basic-info",
        "collection_id": 47,
        "collection_start_time": 1508773513140,
        "msg_timestamp": 1508773513140,
        "collection_end_time": 1508773513441
    },
    "Rows": [
        {
            "Timestamp": 1508773513174,
            "Keys": {
                "name_PIPELINE_EDIT": [
                    "0",
                    "1",
                    "0",
                    "17d0",
                    "0",
                    "0"
                ]
            },
            "Content": {
                "description": "Cisco 100G QSFP28 LR4-S Pluggable Optics Module",
                "firmware-revision": "",
                "hardware-revision": "V01 ",
                "is-field-replaceable-unit": "false",
                "model-name": "QSFP-100G-LR4-S",
                "name": "0/0-Optics0/0/0/0-IDPROM",
                "serial-number": "FNS11111AA1     ",
                "software-revision": "",
                "vendor-type": "1.3.6.1.4.1.9.12.3.1.16.1"
            }
        },
        {
            "Timestamp": 1508773513203,
            "Keys": {
                "name_PIPELINE_EDIT": [
                    "0",
                    "1",
                    "0",
                    "17d5",
                    "0",
                    "0"
                ]
            },
            "Content": {
                "description": "Cisco CFP2 DWDM Pluggable Optics",
                "firmware-revision": "",
                "hardware-revision": "V02 ",
                "is-field-replaceable-unit": "false",
                "model-name": "ONS-CFP2-WDM",
                "name": "0/0-Optics0/0/0/5-IDPROM",
                "serial-number": "OVE1111111A",
                "software-revision": "",
                "vendor-type": "1.3.6.1.4.1.9.12.3.1.16.1"
            }
        }
    ]
}
```

- <b>Client Optics RX and TX power:</b> *Cisco-IOS-XR-controller-optics-oper:optics-oper/optics-ports/optics-port/optics-info.*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Client_TX_RX_optics.png)

With this model you can get information about TX and RX power levels from each lane on each pluggable on the platform. 

- <b>Client Optics Laser Bias Current:</b> *Cisco-IOS-XR-controller-optics-oper:optics-oper/optics-ports/optics-port/optics-info.*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Laser_bias.png)

With this model you can get information about average laser bias current from each lane on each pluggable on the platform. 

- <b>Client Optics RX utilization:</b> *Cisco-IOS-XR-pmengine-oper:performance-management/ethernet/ethernet-ports/ethernet-port/ethernet-current/ethernet-second30/second30-ethers/second30-ether*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/RX_util.png)

This sensor-path gives you highly granular information about RX load on each client port. 

- <b>Pre-FEC and Post-FEC information about NCS1002 line ports:</b> *Cisco-IOS-XR-pmengine-oper:performance-management/otu/otu-ports/otu-port/otu-current/otu-second30/otu-second30fecs/otu-second30fec*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Pre-FEC_BER_nice.png)

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Post-FEC_BER.png)

As it can be seen, you can get almost real time information about Pre-FEC BER on each line port. Data is taken from each 30 seconds interval (the shortest one on NCS1002). Post-FEC BER is expected to be zero on each port (and this can be seen on the graph). 

- <b>Bit errors corrected and uncorrected words:</b> *Cisco-IOS-XR-pmengine-oper:performance-management/otu/otu-ports/otu-port/otu-current/otu-second30/otu-second30fecs/otu-second30fec*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Bit_errors_corrected.png)

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/Uncorrected_words.png)

You can also look at FEC from error bits corrected value and uncorrected words, to have more granular information on how FEC BER is calculated. As with Post-FEC, expectation is that UC-Words number is equal to zero.

- <b>OTN errors on near-end and on far-end:</b> *Cisco-IOS-XR-pmengine-oper:performance-management/otu/otu-ports/otu-port/otu-current/otu-second30/otu-second30otns/otu-second30otn*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/OTN_NE.png)

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/OTN_FE.png)

This model gives information about different OTN parameters for each line port: 
- ES-NE/ES-FE (Error Seconds in the near end / far end)
- ESR-NE/ESR-FE (Error Seconds Ratio on the near end / far end)
- SES-NE/SES-FE (Severely error seconds in the near end / far end)
- SESR-NE/SESR-FE (Severely error seconds ratio in the near end / far end)
- BBE-NE/ BBE-FE (Background block errors in the near end / far end)
- BBER-NE/ BBER-FE (Background block errors in the near end / far end)
Other parameters, such as UAS (Unavailable seconds) and FC (Failure counts) can also be found there.

- <b>OPT and OPR for line ports:</b> *Cisco-IOS-XR-pmengine-oper:performance-management/optics/optics-ports/optics-port/optics-current/optics-second30/optics-second30-optics/optics-second30-optic*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/OPR_OPT.png)

You can get almost real time information about OPT/OPR levels for each line port with this model. Information is taken from each 30 seconds interval.  

- <b>CD / PMD / DGD / OSNR for line ports:</b> *Cisco-IOS-XR-pmengine-oper:performance-management/optics/optics-ports/optics-port/optics-current/optics-second30/optics-second30-optics/optics-second30-optic*

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/CD.png)

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/PMD.png)

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/DGD.png)

![](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/vosipchu/NCS1002_telemetry/OSNR.png)

You can get information about average (also, minimum and maximum, if you want) values for each 30-seconds interval for:
- Chromatic Dispersion
- Polarization Mode Dispersion
- Differential Group Delay
- Optical Signal to Noise ratio

Sensor paths listed here can help you with fast monitoring of different important optical parameters as well as platform itself. Let’s have a look how to configure that.

## NCS1002 Telemetry configuration

There are many possible ways to configure a device to stream the data using telemetry. You can find a good explanation how to do it with CLI configuration [here] (https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) or you can configure Telemetry using [YANG Development Kit (YDK)]( https://developer.cisco.com/site/ydk/), like [here](https://xrdocs.github.io/telemetry/tutorials/2016-08-08-configuring-model-driven-telemetry-with-ydk/) 

Let me first show you an example configuration of models above using CLI. Telemetry configuration will be based on gRPC, Self-Describing GPB and Dial-out Mode.
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
telemetry model-driven
 destination-group DGROUP1
  address-family ipv4 1.1.1.1 port 5432
   encoding self-describing-gpb
   protocol grpc no-tls
  !
  address-family ipv4 10.30.110.38 port 5432
   encoding self-describing-gpb
   protocol grpc no-tls
  !
 !
 sensor-group SGROUP1
  sensor-path Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/attributes/basic-info
  sensor-path Cisco-IOS-XR-plat-chas-invmgr-oper:platform-inventory/racks/rack/slots/slot/cards/card/port-   slots/port-slot/portses/ports/hw-components/hw-component/attributes/basic-info
 !
 sensor-group SGROUP2
  sensor-path Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
  sensor-path Cisco-IOS-XR-alarmgr-server-oper:alarms/brief/brief-card/brief-locations/brief-location/active
 !
 sensor-group SGROUP3
  sensor-path Cisco-IOS-XR-controller-optics-oper-sub1:optics-oper/optics-ports/optics-port/optics-info
  sensor-path Cisco-IOS-XR-pmengine-oper:performance-management/otu/otu-ports/otu-port/otu-current/otu-second30/otu-second30fecs/otu-second30fec
  sensor-path Cisco-IOS-XR-pmengine-oper:performance-management/otu/otu-ports/otu-port/otu-current/otu-second30/otu-second30otns/otu-second30otn
  sensor-path Cisco-IOS-XR-pmengine-oper:performance-management/ethernet/ethernet-ports/ethernet-port/ethernet-current/ethernet-second30/second30-ethers/second30-ether
  sensor-path Cisco-IOS-XR-pmengine-oper:performance-management/optics/optics-ports/optics-port/optics-current/optics-second30/optics-second30-optics/optics-second30-optic
 !
 subscription Sub1
  sensor-group-id SGROUP1 sample-interval 1800000
  destination-id DGROUP1
 !
 subscription Sub2
  sensor-group-id SGROUP2 sample-interval 20000
  destination-id DGROUP1
 !
 subscription Sub3
  sensor-group-id SGROUP3 sample-interval 10000
  destination-id DGROUP1
 !
</code>
</pre>
</div>
MDT configuration using CLI is very straightforward and simple. 

More interesting and powerful way to configure telemetry on NCS1002 is through YDK. If you want to read more about YDK, a lot of details and information can be found [here]( https://communities.cisco.com/community/developer/ydk) with hundreds of examples [here](https://github.com/CiscoDevNet/ydk-py-samples). 
Let's briefly cover major parts of YDK for NCS1002 telemetry configuration for your convenience. For this example I will use native IOS-XR telemetry YANG model. Configuration is based on gRPC Dial-out with self-describing GPB model (the same as it was with CLI). 

Let’s start with configuring destination address/port, encoding and transport protocol:
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
destination_group = telemetry_model_driven.destination_groups.DestinationGroup()
## the name of this destination group
destination_group.destination_id = 'DGROUP'
ipv4_destination = destination_group.ipv4_destinations.Ipv4Destination()
## address and port of the server configuration
ipv4_destination.destination_port = 57500
ipv4_destination.ipv4_address = 10.30.110.38
## define encoding
ipv4_destination.encoding = xr_telemetry_model_driven_cfg.EncodeTypeEnum.self_describing_gpb
protocol = ipv4_destination.Protocol()
## define the transport protocol
protocol.protocol = xr_telemetry_model_driven_cfg.ProtoTypeEnum.grpc
protocol.no_tls = 1
ipv4_destination.protocol = protocol
destination_group.ipv4_destinations.ipv4_destination.append(ipv4_destination)
telemetry_model_driven.destination_groups.destination_group.append(destination_group)
</code>
</pre>
</div>
After destination, encoding and transport configuration, let’s configure a sensor-group with one sensor path:
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
sensor_group = telemetry_model_driven.sensor_groups.SensorGroup()
## the name of this sensor-group
sensor_group.sensor_group_identifier = 'SGROUP'
sensor_group.enable = Empty()        
## define the path you want to be collected
sensor_path = sensor_group.sensor_paths.SensorPath()
sensor_path.telemetry_sensor_path = 'Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary'
sensor_group.sensor_paths.sensor_path.append(sensor_path)   
telemetry_model_driven.sensor_groups.sensor_group.append(sensor_group)
</code>
</pre>
</div>
The final step is to define subscription and configure all together:
<div class="highlighter-rouge">
<pre class="highlight">
<code> 
subscription = telemetry_model_driven.subscriptions.Subscription()
## name of the subscription group
subscription.subscription_identifier = "Sub1"
sensor_profile = subscription.sensor_profiles.SensorProfile()
## attach the sensor-group
sensor_profile.sensorgroupid = 'SGROUP1'
## define the interval
sensor_profile.sample_interval = 20000
subscription.sensor_profiles.sensor_profile.append(sensor_profile) 
sensor_profile = subscription.sensor_profiles.SensorProfile()
## define destination group to be used
destination_profile = subscription.destination_profiles.DestinationProfile()
destination_profile.destination_id = 'DGROUP1'
destination_profile.enable = Empty()
subscription.destination_profiles.destination_profile.append(destination_profile) 
telemetry_model_driven.subscriptions.subscription.append(subscription)
</code>
</pre>
</div>
That’s it! The full version with all sensor paths and two destinations can be found [here]( https://github.com/ios-xr/telemetry-NCS1002/blob/master/YDK-scripts/nc-create-ncs1002-telemetry-native-grpc-out-ydk.py). You can also find there YDK configurations for few other modes, including configurations using OpenConfig models (feel free to modify them and use as you need!)

## NCS1002 telemetry data consumption

At this step we defined the data we want to stream out of an NCS1002. We configured Model-Driven Telemetry with destination, encoding, transport and sampling details. The final step for our task will be to collect and process this data.
There are several ways to achieve this and you're free to use any one that you like. For those of you who have just started looking into telemetry, there are several files at the end to help with faster adoption and testing of things described above.
Steps you need to do have your collector up and running:

1. Clone/download "pipeline" on your server. Pipeline can be found [here](https://github.com/cisco/bigmuddy-network-telemetry-pipeline) and the basic overview of pipeline is [here](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service)

2. Download and install "InfluxDB" on your server. The link for download is [here](https://portal.influxdata.com/downloads)

3. Make sure "pipeline" streams data to "InfluxDB". How to do this is [here](https://xrdocs.github.io/telemetry/tutorials/2017-04-10-using-pipeline-integrating-with-influxdb/). And make sure you don't forget to create a [database!](https://github.com/influxdata/influxdb#create-your-first-database)

4. Install Grafana and add "InfluxDB" database. The link to Grafana is [here](https://grafana.com/grafana/download) or go to their [github link](https://github.com/grafana/grafana)

After these steps are done, you will need to install correct "metrics.json" file that will contain descriptions of models described in this tutorial. For Grafana you will need to have a dashboard configured. As i mentioned above, these files are already prepared for you! You can get "metrics.json" [here](https://github.com/ios-xr/telemetry-NCS1002/blob/master/metrics/metrics.json) and the dashboard for NCS1002 for Grafana can be found [here](https://github.com/ios-xr/telemetry-NCS1002/blob/master/dashboard/NCS1002%20Telemetry.json).

NCS1002 with IOS XR 6.2.2, YDK version 0.5.4, Grafana 4.2.0, InfluxDB v1.0.0 and Python 2.7 were used.

## Conclusion

Telemetry is the modern way to get almost real-time information from your NCS1002 devices. In this tutorial, we covered valuable sensor paths for NCS1002, how to configure them with CLI and YDK and how to process. Try telemetry on NCS1002 today and stay tuned for our next updates. NCS1001 configuration automation tutorials are coming soon!
