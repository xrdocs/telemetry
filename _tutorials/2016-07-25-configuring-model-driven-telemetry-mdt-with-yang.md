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

{% capture "output" %}
Script Output:

```
openconfig-telemetry
Cisco-IOS-XR-telemetry-model-driven-cfg
```
{% endcapture %}


<div class="notice--warning">
{{ output | markdownify }}
</div>

The first model is the [OpenConfig telemetry model](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang) and the second is the XR native telemetry model.  If you look at them in detail, you will notice that the native model closely follows the OpenConfig model.  In fact, the two are more or less functionally equivalent, although the native model will let you do things that are supported by IOS XR but not defined by OpenConfig (like disable TLS).  In this tutorial, I'll focus on openconfig-telemetry, but you could do everything with Cisco-IOS-XR-telemetry-model-driven-cfg as well.

The NETCONF \<get-schema\> operation will give you the contents of the schema but the full YANG output can be really verbose and overwhelming, so I'll pipe the output to the [pyang](https://github.com/mbj4668/pyang) utility for a compact tree view with the following bit of code:

```python
from subprocess import Popen, PIPE, STDOUT

oc = xr.get_schema('Cisco-IOS-XR-telemetry-model-driven-cfg')
p = Popen(['pyang', '-f', 'tree'], stdout=PIPE, stdin=PIPE, stderr=PIPE) 
print(p.communicate(input=oc.data)[0])
```

And voila:  

{% capture "output" %}
Script Output:

```
module: openconfig-telemetry
   +--rw telemetry-system
      +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-id]
      |     +--rw sensor-group-id    -> ../config/sensor-group-id
      |     +--rw config
      |     |  +--rw sensor-group-id?   string
      |     +--ro state
      |     |  +--ro sensor-group-id?   string
      |     +--rw sensor-paths
      |        +--rw sensor-path* [path]
      |           +--rw path      -> ../config/path
      |           +--rw config
      |           |  +--rw path?             string
      |           |  +--rw exclude-filter?   string
      |           +--ro state
      |              +--ro path?             string
      |              +--ro exclude-filter?   string
      +--rw destination-groups
      |  +--rw destination-group* [group-id]
      |     +--rw group-id        -> ../config/group-id
      |     +--rw config
      |     |  +--rw group-id?   string
      |     +--ro state
      |     |  +--ro group-id?   string
      |     +--rw destinations
      |        +--rw destination* [destination-address destination-port]
      |           +--rw destination-address    -> ../config/destination-address
      |           +--rw destination-port       -> ../config/destination-port
      |           +--rw config
      |           |  +--rw destination-address?    inet:ip-address
      |           |  +--rw destination-port?       uint16
      |           |  +--rw destination-protocol?   telemetry-stream-protocol
      |           +--ro state
      |              +--ro destination-address?    inet:ip-address
      |              +--ro destination-port?       uint16
      |              +--ro destination-protocol?   telemetry-stream-protocol
      +--rw subscriptions
         +--rw persistent
         |  +--rw subscription* [subscription-id]
         |     +--rw subscription-id       -> ../config/subscription-id
         |     +--rw config
         |     |  +--rw subscription-id?          uint64
         |     |  +--rw local-source-address?     inet:ip-address
         |     |  +--rw originated-qos-marking?   inet:dscp
         |     +--ro state
         |     |  +--ro subscription-id?          uint64
         |     |  +--ro local-source-address?     inet:ip-address
         |     |  +--ro originated-qos-marking?   inet:dscp
         |     +--rw sensor-profiles
         |     |  +--rw sensor-profile* [sensor-group]
         |     |     +--rw sensor-group    -> ../config/sensor-group
         |     |     +--rw config
         |     |     |  +--rw sensor-group?         -> /telemetry-system/sensor-groups/sensor-group/config/sensor-group-id
         |     |     |  +--rw sample-interval?      uint64
         |     |     |  +--rw heartbeat-interval?   uint64
         |     |     |  +--rw suppress-redundant?   boolean
         |     |     +--ro state
         |     |        +--ro sensor-group?         -> /telemetry-system/sensor-groups/sensor-group/config/sensor-group-id
         |     |        +--ro sample-interval?      uint64
         |     |        +--ro heartbeat-interval?   uint64
         |     |        +--ro suppress-redundant?   boolean
         |     +--rw destination-groups
         |        +--rw destination-group* [group-id]
         |           +--rw group-id    -> ../config/group-id
         |           +--rw config
         |           |  +--rw group-id?   -> ../../../../../../../destination-groups/destination-group/group-id
         |           +--rw state
         |              +--rw group-id?   -> ../../../../../../../destination-groups/destination-group/group-id
         +--rw dynamic
            +--ro subscription* [subscription-id]
               +--ro subscription-id    -> ../state/subscription-id
               +--ro state
               |  +--ro subscription-id?          uint64
               |  +--ro destination-address?      inet:ip-address
               |  +--ro destination-port?         uint16
               |  +--ro destination-protocol?     telemetry-stream-protocol
               |  +--ro sample-interval?          uint64
               |  +--ro heartbeat-interval?       uint64
               |  +--ro suppress-redundant?       boolean
               |  +--ro originated-qos-marking?   inet:dscp
               +--ro sensor-paths
                  +--ro sensor-path* [path]
                     +--ro path     -> ../state/path
                     +--ro state
                        +--ro path?             string
                        +--ro exclude-filter?   string
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

You can spend a lot of time understanding the intricacies of YANG and all the details, but all we really need to know for now is that the model has three major sections:  

- The **destination-group** tells the router where to send telemetry data and how.  
- The **sensor-group** identifies a list of YANG models that the router should stream.  
- The **subscription** ties together the destination-group and the sensor-group.  

Let's see how this works in practice.

## Get-Config

We can use the Cisco-IOS-XR-telemetry-model-driven-cfg model to filter for the telemetry config with the ncclient get_config operation:

```python      
filter = '''<telemetry-system xmlns="http://openconfig.net/yang/telemetry">'''

c = xr.get_config(source='running', filter=('subtree', filter))

print(c)
```

And here's what we get:  

{% capture "output" %}
Script Output:

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
{% endcapture %}

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
<telemetry-system xmlns="http://openconfig.net/yang/telemetry">
   <sensor-groups>
    <sensor-group>
     <sensor-group-id>SGroup1</sensor-group-id>
     <config>
      <sensor-group-id>SGroup1</sensor-group-id>
     </config>
     <sensor-paths>
      <sensor-path>
       <path>Cisco-IOS-XR-ipv4-arp-oper:arp%2fnodes%2fnode%2fentries%2fentry</path>
       <config>
        <path>Cisco-IOS-XR-ipv4-arp-oper:arp%2fnodes%2fnode%2fentries%2fentry</path>
       </config>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <destination-groups>
    <destination-group>
     <group-id>DGroup1</group-id>
     <config>
      <group-id>DGroup1</group-id>
     </config>
     <destinations>
      <destination>
       <destination-address>2001:db8:0:100::b</destination-address>
       <config>
        <destination-address>2001:db8:0:100::b</destination-address>
        <destination-port>5432</destination-port>
        <destination-protocol>grpc</destination-protocol>
       </config>
       <destination-port>5432</destination-port>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
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
