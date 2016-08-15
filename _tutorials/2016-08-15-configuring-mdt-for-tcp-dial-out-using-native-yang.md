---
published: false
date: '2016-08-15 14:05 -0600'
title: Configuring MDT for TCP Dial-out Using Native YANG
---

## Getting the Most out of MDT with Native YANG

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-25-configuring-model-driven-telemetry-mdt-with-yang/), I wrote about how to configure an MDT for gRPC dial-in using the OpenConfig Telemetry YANG model.  In this tutorial, I'll describe how to use the IOS XR Native YANG model to configure MDT with TCP dialout.  I will use [ncclient](https://github.com/ncclient/ncclient) as a simple Python NETCONF client, but you can use whatever client you want.

## The Model

The Cisco IOS XR Native YANG model for telemetry is "Cisco-IOS-XR-telemetry-model-driven-cfg."  It can be used to configure any telemetry feature that IOS XR (unlike the OpenConfig model, which only covers a subset of IOS XR capabilities).

The NETCONF \<get-schema\> operation will give you the contents of the schema but the full YANG output can be really verbose and overwhelming, so I'll pipe the output to the [pyang](https://github.com/mbj4668/pyang) utility for a compact tree view with the following bit of code:

```python

from ncclient import manager
import re
    
xr = manager.connect(host='10.30.111.9', port=830, username='cisco', password='cisco',
                    allow_agent=False,
                    look_for_keys=False,
                    hostkey_verify=False,
                    unknown_host_cb=True)
                    
from subprocess import Popen, PIPE, STDOUT

oc = xr.get_schema('Cisco-IOS-XR-telemetry-model-driven-cfg')
p = Popen(['pyang', '-f', 'tree'], stdout=PIPE, stdin=PIPE, stderr=PIPE) 
print(p.communicate(input=oc.data)[0])

```

And voila:  

{% capture "output" %}

Script Output:

```
module: Cisco-IOS-XR-telemetry-model-driven-cfg
   +--rw telemetry-model-driven
      +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-identifier]
      |     +--rw sensor-paths
      |     |  +--rw sensor-path* [telemetry-sensor-path]
      |     |     +--rw telemetry-sensor-path    string
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
      |     |     |  +--rw protocol!
      |     |     |  |  +--rw protocol        Proto-type
      |     |     |  |  +--rw tls-hostname?   string
      |     |     |  |  +--rw no-tls?         int32
      |     |     |  +--rw encoding?           Encode-type
      |     |     +--rw ipv6* [ipv6-address destination-port]
      |     |        +--rw ipv6-address        xr:Cisco-ios-xr-string
      |     |        +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |        +--rw protocol!
      |     |        |  +--rw protocol        Proto-type
      |     |        |  +--rw tls-hostname?   string
      |     |        |  +--rw no-tls?         int32
      |     |        +--rw encoding?           Encode-type
      |     +--rw destination-id    xr:Cisco-ios-xr-string
      +--rw enable?               empty

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

​

We can use the openconfig-telemetry model to filter for the telemetry config with the ncclient get_config operation:

​

```python      

filter = '''<telemetry-system xmlns="http://openconfig.net/yang/telemetry">'''

​

c = xr.get_config(source='running', filter=('subtree', filter))

​

print(c)

```

​

And here's what we get:  

​

{% capture "output" %}

Script Output:

​

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

​

​

```  

{% endcapture %}

​

<div class="notice--warning">

{{ output | markdownify }}

</div>

​

So what does all that mean to the router?  It breaks down into three parts which you'll recall from the YANG model above:  

​

- The **destination-group** tells the router where to send telemetry data and how.  The absence of a destination-group in the output above alerts us to the fact that this is a dial-in configuration (the collector will initiate the session to the router).

- The **sensor-group** identifies a list of YANG models that the router should stream.  In this case, the router has a sensor-group called "SGroup3" that will send interface statistics data from the OpenConfig Interfaces YANG model.

- The **subscription** ties together the destination-group and the sensor-group.  This router has a subscription name "Sub3" that will send the list of models in SGroup3 at an interval of 30 second (30000 milleseconds).  

​

If you read the [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) on configuring MDT with CLI, you might recognize this as the same as the gRPC dial-in configuration described there.  If you missed that thrilling installment, the XML above is the YANG equivalent of this CLI:  

​

{% capture "output" %}

CLI Output:

​

```

telemetry model-driven

 sensor-group SGroup3

  sensor-path openconfig-interfaces:interfaces/interface

 !

 subscription Sub3

  sensor-group-id SGroup3 sample-interval 30000

 !  

``` 

​

{% endcapture %}

​

<div class="notice--info">

{{ output | markdownify }}

</div>

​

## Edit-Config

​

So let's say we want to add a second model to SGroup3 (Cisco-IOS-XR-ipv4-arp-oper).  We can do that with the following NETCONF operations:

​

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

​

xr.edit_config(edit_data, target='candidate', format='xml')

xr.commit()

```

​

If we do a get-config operation again:  

​

```python

c = xr.get_config(source='running', filter=('subtree', filter))

print(c)

```

​

... we'll see that SGroup3 has the new addition.  

​

​

{% capture "output" %}

Script Output:

​

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

​

```

{% endcapture %}

​

<div class="notice--warning">

{{ output | markdownify }}

</div>

​

And if you need some CLI to reassure yourself that it worked, here it is:

​

{% capture "output" %}

CLI Output:

​

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

​

<div class="notice--info">

{{ output | markdownify }}

</div>

​

## Conclusion

Armed with the examples in this blog and a understanding of the telemetry YANG model, you should now be able to use YANG configuration models to configure the router to stream YANG models with the operational data you want.  How's that for model-driven programmability?

​

Prose

    Prose
    About
    Developers
    Language
    Logout


