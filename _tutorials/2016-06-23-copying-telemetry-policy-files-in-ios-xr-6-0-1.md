---
published: true
date: '2016-06-23 13:03 -0400'
title: Copying Telemetry Policy Files in IOS XR 6.0.1
author: Shelly Cadora
excerpt: How to telemetry policy files to IOS XR in 6.0.1
tags:
  - iosxr
  - telemetry
---
Due to some general security improvements in 6.0.1, it's not possible to sftp/scp files directly to the /telemetry/policies directory from the outside.  If you try, you might see something like this:  

{% capture include-text %}
RP/0/RP0/CPU0:Sun601#**run**
[xr-vm_node0_RP0_CPU0:~]$**sftp scadora@172.30.8.11** 
Connecting to 172.30.8.11...  
Password:  
sftp&gt;**get /tftpboot/BasicPolicy.policy /telemetry/policies/BasicPolicy.policy**  
  RP/0/RP0/CPU0:Jun 23 16:08:00.870 : sftp[69048]: %SECURITY-SSHD-3-ERR_GENERAL : Cannot overwrite system files  
sftp&gt;
{% endcapture %}  

<div class="highlighter-rouge">
  {{ include-text | markdownify }}
</div>

The restriction on the /telemetry/policies directory will be lifted in 6.0.2, but in the meantime you can work around this by copying files to disk0: and then doing a local copy to the proper directory as follows:  

>
RP/0/RP0/CPU0:Sun601#**run**
[xr-vm_node0_RP0_CPU0:~]$**sftp scadora@172.30.8.11**
Connecting to 172.30.8.11...  
Password:  
sftp&gt; **get /tftpboot/BasicPolicy.policy /disk0:/BasicPolicy.policy** 
  Transferred 469 Bytes  
  469 bytes copied in 0 sec (0)bytes/sec  
sftp&gt; **quit**  
[xr-vm_node0_RP0_CPU0:~]$**cp /disk0:/BasicPolicy.policy /telemetry/policies**
[xr-vm_node0_RP0_CPU0:~]$

