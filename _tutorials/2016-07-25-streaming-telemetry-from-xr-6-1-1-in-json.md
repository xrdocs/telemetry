---
published: false
date: '2016-07-25 10:10 +0300'
title: Streaming telemetry from XR 6.1.1 in JSON
author: Mike Korshunov
excerpt: >-
  Tutorial about configuration policy driven telemetry on XR and using ELK stack
  to grab data.
tags:
  - vagrant
  - iosxr
  - cisco
---
## About

Think of Streaming Telemetry as a policy driven mechanism for pushing data off the box. Data can be pushed very frequent (30s, 10s and etc). In this tutorial PDT (policy driven telemetry) will be configured.

## Prerequsition

For this tutorial will be used old setup, which consist of devbox instance (any linux) and IOS-XRv instance. 

![setup](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/mkorshun/telemetry_setup.png){: .align-center}
{: .notice}

## Configure XR

Following strings should be applied to rtr. Data from router would be streamed in json format and tcp will be used for transport. Destination IP would be devbox IP address. 

```
telemetry
 encoder json
  policy group FirstGroup
   policy test
   transport tcp
   !
   destination ipv4 10.1.1.10 port 2103
  !
 !
!
```

In configuration we used "policy test", so we will need file with same name on XR Linux shell. Create it and describe simple policy for telemetry. Period - how frequently data is sent. In "Paths" you can specify list of metrics, which you want to sent. 

```
RP/0/RP0/CPU0:xr#run
Tue Jul 19 10:17:23.754 UTC
[xr-vm_node0_RP0_CPU0:~]$cat /telemetry/policies/test.policy
{
        "Name": "test",
        "Metadata": {
                "Version": 25,
                "Description": "This is a sample policy to demonstrate the syntax",
                "Comment": "This is the first draft",
                "Identifier": "<data that may be sent by the encoder to the mgmt stn>"
        },
        "CollectionGroups": {
                "FirstGroup": {
                        "Period": 30,
                        "Paths": [
                                "RootOper.InfraStatistics.Interface([*]).Latest.GenericCounters"
                        ]
                }
        }
}

```

Verify, that policy is applied:

```
RP/0/RP0/CPU0:xr#show telemetry policies
Tue Jul 19 10:29:41.194 UTC

test
  Filename:             test.policy
  Version:              25
  Description:          This is a sample policy to demonstrate the syntax
  Status:               Active
  CollectionGroup: FirstGroup
    Cadence:              30.0s
    Total collections:    24
    Latest collection:    2016-07-19 10:29:22
    Min total time:       0.005s
    Max total time:       0.088s
    Avg total time:       0.012s
    Collection errors:    0
    Missed collections:   0
    +----------------------------------------------+---------+---------+------+
    | Path                                         | Avg (s) | Max (s) | Err  |
    +----------------------------------------------+---------+---------+------+
    | RootOper.InfraStatistics.Interface([*]).Late |   0.012 |   0.088 |    0 |
    +----------------------------------------------+---------+---------+------+
```

Good, first part is done and router streaming the data. Now we should configure data receiver. 

## Configure devbox

Make sure, you have at least 2gb of RAM, allocated to VM
{: .notice--warning}


To do it add following strings to Vagrantfile. If VM was runned previously, use command "vagrant reload"  to reapply configuration.

```
Vagrant.configure(2) do |config|
  config.vm.define "devbox" do |devbox|
    config.vm.provider "virtualbox" do |vb|
      vb.customize ["modifyvm", :id, "--memory", "2048"]
    end
	## Omitted configuration
```

Free RAM can be checked with command ```free -m```

Good to configure port forwarding for Kibana web interface
{: .notice--info}

To do it, add one string to Vagrantfile:

```
    devbox.vm.network "forwarded_port", id: "kibana", guest: 5601, host: 5601, auto_correct: true
```

### Install soft on devbox

To run this demo 

- [Docker](https://docs.docker.com/engine/installation/linux/ubuntulinux/)
- protobuf-compiler package.  Install it via ```apt-get install protobuf-compiler```
- [ELK stack](https://github.com/cisco/bigmuddy-network-telemetry-stacks)
- [Collector](https://github.com/cisco/bigmuddy-network-telemetry-collector)


### Verify data is coming with collector

Let's clone the repo:

```
git clone https://github.com/cisco/bigmuddy-network-telemetry-collector.git
```

To start collector, we should use devbox IP address and destination port from XR telemetry policy.  

```
cd bigmuddy-network-telemetry-collector/
chmod 777 telemetry_receiver.py
./telemetry_receiver.py --ip-address 10.1.1.10 --port 2103 
/sw/packages/protoc/current/google/include/: warning: directory does not exist.
google/protobuf/descriptor.proto: File not found.
cisco.proto: Import "google/protobuf/descriptor.proto" was not found or had errors.
cisco.proto:39:8: "google.protobuf.MessageOptions" is not defined.
cisco.proto:43:8: "google.protobuf.FieldOptions" is not defined.
cisco.proto:47:8: "google.protobuf.FileOptions" is not defined.
Compiled cisco.proto
Waiting for TCP connection
Waiting for UDP message
Getting TCP message
  Message Type: JSON (2))
  Flags: None
  Length: 3106
Decoding message

CollectionStartTime: Wed Jul 20 03:13:22 2016 (930ms)
CollectionID: 1716
CollectionEndTime: Wed Jul 20 03:13:22 2016 (952ms)
Version: 25
Policy: test
Path: RootOper.InfraStatistics.Interface.Latest.GenericCounters
Identifier: <data that may be sent by the encoder to the mgmt stn>
Data: {
  RootOper {
    InfraStatistics {
      Interface (3 items - displaying first entry only) [
        [0]
          InterfaceName: Null0
          Latest {
            GenericCounters {
              InputOverruns: 0
              ParityPacketsReceived: 0
              MulticastPacketsSent: 0
              MulticastPacketsReceived: 0
              InputIgnoredPackets: 0
              OutputQueueDrops: 0
              SecondsSinceLastClearCounters: 0
              Applique: 0
              SecondsSincePacketSent: 4294967295
              PacketsSent: 0
              OutputBuffersSwappedOut: 0
              CollectionTime: Wed Jul 20 03:13:22 2016 (950ms)
              GiantPacketsReceived: 0
              SecondsSincePacketReceived: 4294967295
              InputErrors: 0
              BytesReceived: 0
              LastDiscontinuityTime: 1468922633
              OutputErrors: 0
              CarrierTransitions: 0
              LastDataTime: 1468984401
              CRCErrors: 0
              OutputDrops: 0
              PacketsReceived: 0
              InputQueueDrops: 0
              RuntPacketsReceived: 0
              InputAborts: 0
              InputDrops: 0
              ThrottledPacketsReceived: 0
              Resets: 0
              FramingErrorsReceived: 0
              BroadcastPacketsSent: 0
              OutputUnderruns: 0
              OutputBufferFailures: 0
              BytesSent: 0
              UnknownProtocolPacketsReceived: 0
              BroadcastPacketsReceived: 0
              AvailabilityFlag: 0
            }
          }
      ]
    }
  }

```

Hooray! Messages are coming, let's move to ELK stack. 

### Build and run ELK stack

Clone the repo from github:

```
git clone https://github.com/cisco/bigmuddy-network-telemetry-stacks.git
cd stack_elk
```

Modify 2 strings in **ls_temetry.conf**: 
`xform => "raw"` and add string `wire_format => 2`

```
$ cat ls_telemetry.conf
#
# cisco telemetry codec
#
# This file is staged into the ../staging directory, updated according
# to setup in ../environment, and copied into the host volume for
# logstash (under $LOGSTASH_VOLUME/conf.d). File can be edited there
# but will be overwritten if container is rebuilt.
#
#
input {
    tcp {
        port => TELEMETRYPORTTCP_PLACEHOLDER
        codec => telemetry {
            xform => "raw"
            xform_flat_delimeter => "~"
            xform_flat_keys => [
                'interfaces', 'RootOper~Interfaces~(?<InterfaceName>.*)',
                'ipslastats', 'RootOper~IPSLA~OperationData~(?<operation>\d+)~Statistics',
                'ipslacommon', 'RootOper~IPSLA~OperationData~(?<operation>\d+)~Common',
                'counters', 'RootOper~InfraStatistics~(?<InterfaceName>.*)~Latest~GenericCounters',
                'datarates', 'RootOper~InfraStatistics~(?<InterfaceName>.*)~Latest~DataRate',
                'ipaddress', 'RootOper~IPV4ARM~Addresses~(?<VRFName>.*)~(?<InterfaceName>.*)',
                'labelcontext', 'RootOper~MPLS_LSD~(?<label>\d+)',
                'labelspecial', 'RootOper~MPLS_LSD~Label',
                'fiblabel', 'RootOper~FIB_MPLS~(?<card>.*)~LabelFIB~(?<label>\d+)~(?<EOS>EOS\d)',
                'fibprefixv4', 'RootOper~FIB~(?<card>.*)~IPv4~(?<vrf>.*)~(?<pfx>.*)~(?<len>\d+)',
                'fibprefixv6', 'RootOper~FIB~(?<card>.*)~IPv6~(?<vrf>.*)~(?<pfx>.*)~(?<len>\d+)',
                'fibprefixmpls', 'RootOper~FIB~(?<card>.*)~MPLS~(?<vrf>.*)~(?<pfx>.*)~(?<len>\d+)',
                'ribprefix', 'RootOper~RIB~(?<vrf>.*)~IPv4~Unicast~default~(?<pfx>.*)~(?<len>\d+)~-~-',
                'mplstesummary', 'RootOper~MPLS_TE~Tunnels~Summary',
                'mplstetopology', 'RootOper~MPLS_TE~Topology',
                'mplsteautoroute', 'RootOper~MPLS_TE~AnnounceTunnelsInfo~AutorouteAnnounceTable~(?<RouterID>.*)~(?<Protocol>.*)~(?<Area>\d+)~(?<Instance>.*)']
        wire_format => 2
        }

    }

```


Build the ELK stack:

```
sudo COLLECTOR=10.1.1.10 ./stack_build
```

Run the ELK stack:

```
sudo ./stack_run
--------------------------------------------------------------

%%% Running stack_run on stack_elk  - Mon Jul 25 06:45:05 UTC 2016 %%%

 Please 'tail -f log/stack_run.log' to watch the action in detail

--------------------------------------------------------------
%%-script:./stack_run:LOG: Collector for stack_elk at 10.1.1.10
%%-script:./stack_run:LOG: Run stack containers
%%-script:./stack_run:LOG: Executing: docker ps -a
CONTAINER ID        IMAGE                 COMMAND                  CREATED                  STATUS                  PORTS                    NAMES
fc5f98486b29        logstash:2.3.1        "/bin/sh -c '/start.s"   Less than a second ago   Up Less than a second                            stack_elk_logstash
7f46aa4911ad        kibana:4.5.0          "/bin/sh -c '/start.s"   Less than a second ago   Up Less than a second   0.0.0.0:5601->5601/tcp   stack_elk_kibana
af1d8852c847        elasticsearch:2.3.1   "/docker-entrypoint.s"   1 seconds ago            Up Less than a second                            stack_elk_elasticsearch
--------------------------------------------------------------

%%% Ran stack_run on stack_elk successfully - Mon Jul 25 06:45:06 UTC 2016 %%%

--------------------------------------------------------------

Telemetry streams can be pointed at collector, and data viewed in kibana:

Collector       @ 10.1.1.10
Streams         @ UDP/2103 supporting gprotobuf or JSON
Streams         @ TCP/2103 supporting compressed JSON
Kibana          @ http://10.1.1.10:5601

Note: if host has multiple addresses, use an address which is reachable from source (router or browser as the case may be).

--------------------------------------------------------------

```

Great, stack is running, let's verify message receival in Kibana. To do it go to: [http://127.0.0.1:5601/](http://127.0.0.1:5601/)

![kibana](https://xrdocs.github.io/xrdocs-images/assets/tutorial-images/mkorshun/telemetry_kibana.PNG){: .align-center}
{: .notice}

Congratulation! Messages are coming from XR, you will see hit counter increased over time.{: .notice--success}  
 
