---
published: true
date: '2016-07-25 11:42 -0600'
title: Configuring Model-Driven Telemetry (MDT) with YANG
author: Shelly Cadora
excerpt: Describes how to configure MDT with YANG models
tags:
  - iosxr
position: hidden
---
## Model-Driven Configuration for Model-Driven Telemetry

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I wrote about how to configure MDT using CLI.  But if the router is using YANG models to structure the operational data it streams, shouldn't we also be able to use models to configure the telemetry feature itself?  The answer is yes!  In this tutorial, we'll look at the XR YANG model for telemetry and how to configure it.  I will use [ncclient](https://github.com/ncclient/ncclient) as a simple Python NETCONF client, but you can use whatever client you want.  

## The Models

Let's start with a quick look at the NETCONF capabilities list from IOS XR 6.1.1.  This bit of code:

```python
from ncclient import manager
import re
    
xr = manager.connect(host='10.30.111.9', port=830, username='cisco', password='cisco',
                    allow_agent=False,
                    look_for_keys=False,
                    hostkey_verify=False,
                    unknown_host_cb=True)

for c in xr.server_capabilities:
    model = re.search('module=([^&]*telemetry[^&]*)&', c)
    if model is not None:
        print model.group(1)     
```


...tells us that there are two models for telemetry configuration:  

```
openconfig-telemetry
Cisco-IOS-XR-telemetry-model-driven-cfg
```

The first model is the [OpenConfig telemetry model](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang) and the second is the XR Native telemetry model.  If you look at them in detail, you will notice that the native model closely follows the OpenConfig model.  In this tutorial, I'll use Cisco-IOS-XR-telemetry-model-driven-cfg, but the two are functionally equivalent.

The NETCONF \<get-schema\> operation will give you the contents of the schema but the full YANG output can be really verbose and overwhelming, so I'll pipe the output to the [pyang](https://github.com/mbj4668/pyang) utility for a compact tree view with the following bit of code:

```python
from subprocess import Popen, PIPE, STDOUT

c = xr.get_schema('Cisco-IOS-XR-telemetry-model-driven-cfg')

p = Popen(['pyang', '-f', 'tree'], stdout=PIPE, stdin=PIPE, stderr=PIPE) 
print(p.communicate(input=c.data)[0])
```

And voila:  

```
module: Cisco-IOS-XR-telemetry-model-driven-cfg
   +--rw telemetry-model-driven
      +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-identifier]
      |     +--rw sensor-paths
      |     |  +--rw sensor-path* [telemetry-sensor-path]
      |     |     +--rw telemetry-sensor-path    xr:Cisco-ios-xr-string
      |     +--rw enable?                    empty
      |     +--rw sensor-group-identifier    xr:Cisco-ios-xr-string
      +--rw subscriptions
      |  +--rw subscription* [subscription-identifier]
      |     +--rw source-address!
      |     |  +--rw address-family    Af
      |     |  +--rw ip-address?       inet:ipv4-address-no-zone
      |     |  +--rw ipv6-address?     string
      |     +--rw sensor-profiles
      |     |  +--rw sensor-profile* [sensorgroupid]
      |     |     +--rw sample-interval?      uint32
      |     |     +--rw heartbeat-interval?   uint32
      |     |     +--rw supress-redundant?    empty
      |     |     +--rw sensorgroupid         xr:Cisco-ios-xr-string
      |     +--rw destination-profiles
      |     |  +--rw destination-profile* [destination-id]
      |     |     +--rw enable?           empty
      |     |     +--rw destination-id    xr:Cisco-ios-xr-string
      |     +--rw source-qos-marking?        uint32
      |     +--rw subscription-identifier    xr:Cisco-ios-xr-string
      +--rw destination-groups
      |  +--rw destination-group* [destination-id]
      |     +--rw destinations
      |     |  +--rw destination* [address-family]
      |     |     +--rw address-family    Af
      |     |     +--rw ipv4* [ipv4-address destination-port]
      |     |     |  +--rw ipv4-address        inet:ip-address-no-zone
      |     |     |  +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |     |  +--rw encoding?           Encode-type
      |     |     |  +--rw protocol?           Proto-type
      |     |     +--rw ipv6* [ipv6-address destination-port]
      |     |        +--rw ipv6-address        xr:Cisco-ios-xr-string
      |     |        +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |        +--rw encoding?           Encode-type
      |     |        +--rw protocol?           Proto-type
      |     +--rw destination-id    xr:Cisco-ios-xr-string
      +--rw enable?               empty
```

You can spend a lot of time understanding the intricacies of YANG and all the details, but all we really need to know for now is that the model has three major sections:  

- The **destination-group** tells the router where to send telemetry data and how.  
- The **sensor-group** identifies a list of YANG models that the router should stream.  
- The **subscription** ties together the destination-group and the sensor-group.  

Let's see how this works in practice.

## Get-Config

We can use the Cisco-IOS-XR-telemetry-model-driven-cfg model to filter for the telemetry config with the ncclient get_config operation:

```python      
filter = '''<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">'''

c = xr.get_config(source='running', filter=('subtree', filter))

print(c)
```

And here's what we get:  

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:e884966d-6d41-4ca9-8a47-f4bccdf5af68" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination>
       <address-family>ipv4</address-family>
       <ipv4>
        <ipv4-address>172.30.8.4</ipv4-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>tcp</protocol>
       </ipv4>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <enable></enable>
     <sensor-paths>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-infra-statsd-oper:infra-statistics%2finterfaces%2finterface%2flatest%2fgeneric-counters</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <enable></enable>
   <subscriptions>
    <subscription>
     <subscription-identifier>Sub1</subscription-identifier>
     <sensor-profiles>
      <sensor-profile>
       <sensorgroupid>SGroup1</sensorgroupid>
       <sample-interval>30000</sample-interval>
      </sensor-profile>
     </sensor-profiles>
     <destination-profiles>
      <destination-profile>
       <destination-id>DGroup1</destination-id>
       <enable></enable>
      </destination-profile>
     </destination-profiles>
    </subscription>
   </subscriptions>
  </telemetry-model-driven>
 </data>
</rpc-reply>

```

So what does all that mean to the router?  It breaks down into three parts which you'll recall from the YANG model above:  

- The **destination-group** tells the router where to send telemetry data and how.  If you parse the XML above, you'll see that the router has a destination group named "DGroup 1" that goes to 172.30.8.4 port 5432 using the self-describing GPB encoding.
- The **sensor-group** identifies a list of YANG models that the router should stream.  In this case, the router has a sensor-group called "SGroup1" that will send interface statistics data from the Cisco-IOS-XR-infra-statsd-oper YANG model.
- The **subscription** ties together the destination-group and the sensor-group.  This router has a subscription name "Sub1" that will send the list of models in SGroup1 to the destinations in DGroup1 at an interval of 30 second (30000 milleseconds).  

If you read the [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) on configuring MDT with CLI, you might recognize this as the same as the TCP Dial-Out configuration described there.  If you missed that thrilling installment, the XML above is the YANG equivalent of this CLI:

```
telemetry model-driven  
 destination-group DGroup1  
   address family ipv4 172.30.8.4 port 5432  
   encoding self-describing-gpb  
   protocol tcp
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !  
 subscription Sub1  
  sensor-group-id SGroup1 sample-interval 30000  
  destination-id DGroup1   
``` 

## Edit-Config

So let's say we want to add a second destination (to 2001:db8:0\:100::b) to DGroup1 and a second model to SGroup1 (Cisco-IOS-XR-ipv4-arp-oper).  We can do that with the following NETCONF operations:

```python
edit_data = '''
<config>
<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination>
       <address-family>ipv6</address-family>
       <ipv6>
        <ipv6-address>2001:db8:0:100::b</ipv6-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>grpc</protocol>
       </ipv6>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <sensor-paths>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-ipv4-arp-oper:arp%2fnodes%2fnode%2fentries%2fentry</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
  </telemetry-model-driven>
</config>
'''

xr.edit_config(edit_data, target='candidate', format='xml')
xr.commit()
```

If we do a get-config operation again:  

```python
c = xr.get_config(source='running', filter=('subtree', filter))
print(c)
```

... we'll see that SGroup1 and DGroup1 have the new additions.  

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:7fcd69e3-1e4b-457d-b41d-9e2c82faf76a" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination>
       <address-family>ipv4</address-family>
       <ipv4>
        <ipv4-address>172.30.8.4</ipv4-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>tcp</protocol>
       </ipv4>
      </destination>
      <destination>
       <address-family>ipv6</address-family>
       <ipv6>
        <ipv6-address>2001:db8:0:100::b</ipv6-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>grpc</protocol>
       </ipv6>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <enable></enable>
     <sensor-paths>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-ipv4-arp-oper:arp%2fnodes%2fnode%2fentries%2fentry</telemetry-sensor-path>
      </sensor-path>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-infra-statsd-oper:infra-statistics%2finterfaces%2finterface%2flatest%2fgeneric-counters</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <enable></enable>
   <subscriptions>
    <subscription>
     <subscription-identifier>Sub1</subscription-identifier>
     <sensor-profiles>
      <sensor-profile>
       <sensorgroupid>SGroup1</sensorgroupid>
       <sample-interval>30000</sample-interval>
      </sensor-profile>
     </sensor-profiles>
     <destination-profiles>
      <destination-profile>
       <destination-id>DGroup1</destination-id>
       <enable></enable>
      </destination-profile>
     </destination-profiles>
    </subscription>
   </subscriptions>
  </telemetry-model-driven>
 </data>
</rpc-reply>
```

And if you need some CLI to reassure yourself that it worked, here it is:

```
RP/0/RP0/CPU0:SunC#show run telemetry model-driven
Mon Jul 25 19:27:26.632 UTC
telemetry model-driven
 destination-group DGroup1
  address family ipv4 172.30.8.4 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
  address family ipv6 2001:db8:0:100::b port 5432
   encoding self-describing-gpb
   protocol grpc
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-ipv4-arp-oper:arp/nodes/node/entries/entry
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription Sub1
  sensor-group-id SGroup1 sample-interval 30000
  destination-id DGroup1
 !
!
```

## Conclusion
Armed with the examples in this blog and a understanding of the telemetry YANG model, you should now be able to use YANG configuration models to configure the router to stream YANG models with the operational data you want.  How's that for model-driven programmability?
