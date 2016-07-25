---
published: false
date: '2016-07-25 11:42 -0600'
title: Configuring Model-Driven Telemetry (MDT) with YANG
author: Shelly Cadora
excerpt: Describes how to configure MDT with YANG models
tags:
  - iosxr
---
## Model-Driven Configuration for Model-Driven Telemetry

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/), I wrote about how to configure MDT using CLI.  But if the router is using YANG models to structure the operational data it stream, shouldn't we also be able to use models to configure the telemetry feature itself?  The answer is yes!  In this tutorial, we'll look at the XR YANG model for telemetry and how to configure it.  I will use [ncclient](https://github.com/ncclient/ncclient) as a simple Python NETCONF client, but you can use whatever client you want.  

## The Models

Let's start with a quick look at the NETCONF capabilities list from IOS XR 6.1.1.  This bit of code:

```
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

...will return two models for telemetry:  

```
openconfig-telemetry
Cisco-IOS-XR-telemetry-model-driven-cfg
```

The first model is the [OpenConfig telemetry model](https://github.com/openconfig/public/blob/master/release/models/telemetry/openconfig-telemetry.yang) and the second is the XR Native telemetry model.  If you look at them in detail, you will notice that the native model closely follows the OpenConfig model.  In this tutorial, I'll use Cisco-IOS-XR-telemetry-model-driven-cfg, but the two are functionally equivalent.