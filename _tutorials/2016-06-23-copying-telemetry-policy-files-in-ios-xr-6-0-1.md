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

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:Sun601#<mark>bash</mark>
[xr-vm_node0_RP0_CPU0:~]$<mark>sftp scadora@172.30.8.11</mark> 
Connecting to 172.30.8.11...  
Password:  
sftp&gt;<mark>get /tftpboot/BasicPolicy.policy /telemetry/policies/BasicPolicy.policy</mark> 
  RP/0/RP0/CPU0:Jun 23 16:08:00.870 : sftp[69048]: %SECURITY-SSHD-3-ERR_GENERAL : Cannot   
  overwrite system files  
sftp&gt;
</code>
</pre>
</div>


The restriction on the /telemetry/policies directory will be lifted in 6.0.2, but in the meantime you can work around this by copying files to disk0: and then doing a local copy to the proper directory as follows:  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:Sun601#<mark>bash</mark>
[xr-vm_node0_RP0_CPU0:~]$<mark>sftp scadora@172.30.8.11</mark>
Connecting to 172.30.8.11...  
Password:  
sftp&gt; <mark>get /tftpboot/BasicPolicy.policy /disk0:/BasicPolicy.policy</mark>
  Transferred 469 Bytes  
  469 bytes copied in 0 sec (0)bytes/sec  
sftp&gt; <mark>quit</mark>  
[xr-vm_node0_RP0_CPU0:~]$<mark>cp /disk0:/BasicPolicy.policy /telemetry/policies</mark>
[xr-vm_node0_RP0_CPU0:~]$
</code>
</pre>
</div>
