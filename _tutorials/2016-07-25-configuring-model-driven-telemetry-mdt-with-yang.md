---
published: true
date: '2016-07-25 11:42 -0600'
title: Configuring Model-Driven Telemetry (MDT) with OpenConfig YANG
author: Shelly Cadora
excerpt: Describes how to configure MDT with YANG models
tags:
  - iosxr
position: top
---
## Model-Driven Configuration for Model-Driven Telemetry

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I wrote about how to configure MDT using CLI.  But if the router is using YANG models to structure the operational data it streams, shouldn't we also be able to use models to configure the telemetry feature itself?  The answer is yes!  In this tutorial, we'll look at the OpenConfig YANG model for telemetry and how to configure it.  I will use [ncclient](https://github.com/ncclient/ncclient) as a simple Python NETCONF client, but you can use whatever client you want.  

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

The first model is the [OpenConfig telemetry model](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang) and the second is the XR native telemetry model.  If you look at them in detail, you will notice that the native model closely follows the OpenConfig model, although the native model will let you do things that are supported by IOS XR but not defined by this version of OpenConfig (like disabling TLS or enabling dial-out).  In this tutorial, I'll focus on openconfig-telemetry, but you could do everything with Cisco-IOS-XR-telemetry-model-driven-cfg as well.

The NETCONF \<get-schema\> operation will give you the contents of the schema but the full YANG output can be really verbose and overwhelming, so I'll pipe the output to the [pyang](https://github.com/mbj4668/pyang) utility for a compact tree view with the following bit of code:

```python
from subprocess import Popen, PIPE, STDOUT

oc = xr.get_schema('openconfig-telemetry')
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

- The **destination-group** tells the router where to send telemetry data and how. Only needed for dial-out configuration.  
- The **sensor-group** identifies a list of YANG models that the router should stream.  
- The **subscription** ties together the destination-group and the sensor-group.  

Let's see how this works in practice.

## Get-Config

We can use the openconfig-telemetry model to filter for the telemetry config with the ncclient get_config operation:

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
<rpc-reply message-id="urn:uuid:939c718e-81ee-43ec-9733-565aa53fedb2" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-system xmlns="http://openconfig.net/yang/telemetry">
   <sensor-groups>
    <sensor-group>
     <sensor-group-id>SGroup3</sensor-group-id>
     <config>
      <sensor-group-id>SGroup3</sensor-group-id>
     </config>
     <sensor-paths>
      <sensor-path>
       <path>openconfig-interfaces:interfaces/interface</path>
       <config>
        <path>openconfig-interfaces:interfaces/interface</path>
       </config>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <subscriptions>
    <persistent>
     <subscription>
      <subscription-id>Sub3</subscription-id>
      <config>
       <subscription-id>Sub3</subscription-id>
      </config>
      <sensor-profiles>
       <sensor-profile>
        <sensor-group>SGroup3</sensor-group>
        <config>
         <sensor-group>SGroup3</sensor-group>
         <sample-interval>30000</sample-interval>
        </config>
       </sensor-profile>
      </sensor-profiles>
     </subscription>
    </persistent>
   </subscriptions>
  </telemetry-system>
 </data>
</rpc-reply>


```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

So what does all that mean to the router?  It breaks down into three parts which you'll recall from the YANG model above:  

- The **destination-group** tells the router where to send telemetry data and how.  The absence of a destination-group in the output above alerts us to the fact that this is a dial-in configuration (the collector will initiate the session to the router).
- The **sensor-group** identifies a list of YANG models that the router should stream.  In this case, the router has a sensor-group called "SGroup3" that will send interface statistics data from the OpenConfig Interfaces YANG model.
- The **subscription** ties together the destination-group and the sensor-group.  This router has a subscription name "Sub3" that will send the list of models in SGroup3 at an interval of 30 second (30000 milleseconds).  

If you read the [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) on configuring MDT with CLI, you might recognize this as the same as the gRPC dial-in configuration described there.  If you missed that thrilling installment, the XML above is the YANG equivalent of this CLI:  

{% capture "output" %}
CLI Output:

```
telemetry model-driven
 sensor-group SGroup3
  sensor-path openconfig-interfaces:interfaces/interface
 !
 subscription Sub3
  sensor-group-id SGroup3 sample-interval 30000
 !  
``` 

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>

## Edit-Config

So let's say we want to add a second model to SGroup3 (Cisco-IOS-XR-ipv4-arp-oper).  We can do that with the following NETCONF operations:

```python
edit_data = '''
<config>
<telemetry-system xmlns="http://openconfig.net/yang/telemetry">
   <sensor-groups>
    <sensor-group>
     <sensor-group-id>SGroup3</sensor-group-id>
     <sensor-paths>
      <sensor-path>
       <config>
        <path>Cisco-IOS-XR-ipv4-arp-oper:arp/nodes/node/entries/entry</path>
       </config>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
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

... we'll see that SGroup3 has the new addition.  


{% capture "output" %}
Script Output:

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:abd0a7ee-5f06-4754-b2a3-dae6e3d797aa" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-system xmlns="http://openconfig.net/yang/telemetry">
   <sensor-groups>
    <sensor-group>
     <sensor-group-id>SGroup3</sensor-group-id>
     <config>
      <sensor-group-id>SGroup3</sensor-group-id>
     </config>
     <sensor-paths>
      <sensor-path>
       <path>openconfig-interfaces:interfaces/interface</path>
       <config>
        <path>openconfig-interfaces:interfaces/interface</path>
       </config>
      </sensor-path>
      <sensor-path>
       <path>Cisco-IOS-XR-ipv4-arp-oper:arp/nodes/node/entries/entry</path>
       <config>
        <path>Cisco-IOS-XR-ipv4-arp-oper:arp/nodes/node/entries/entry</path>
       </config>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <subscriptions>
    <persistent>
     <subscription>
      <subscription-id>Sub3</subscription-id>
      <config>
       <subscription-id>Sub3</subscription-id>
      </config>
      <sensor-profiles>
       <sensor-profile>
        <sensor-group>SGroup3</sensor-group>
        <config>
         <sensor-group>SGroup3</sensor-group>
         <sample-interval>30000</sample-interval>
        </config>
       </sensor-profile>
      </sensor-profiles>
     </subscription>
    </persistent>
   </subscriptions>
  </telemetry-system>
 </data>
</rpc-reply>

```
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

And if you need some CLI to reassure yourself that it worked, here it is:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:SunC#show run telemetry model-driven
Mon Aug  8 20:09:57.149 UTC
telemetry model-driven
 sensor-group SGroup3
  sensor-path openconfig-interfaces:interfaces/interface
  sensor-path Cisco-IOS-XR-ipv4-arp-oper:arp/nodes/node/entries/entry
 !
 subscription Sub3
  sensor-group-id SGroup3 sample-interval 30000
 !
!
```
{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>

## Conclusion
Armed with the examples in this blog and a understanding of the telemetry YANG model, you should now be able to use YANG configuration models to configure the router to stream YANG models with the operational data you want.  How's that for model-driven programmability?
