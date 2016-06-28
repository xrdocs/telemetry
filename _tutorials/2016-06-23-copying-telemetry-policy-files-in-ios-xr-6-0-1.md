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

RP/0/RP0/CPU0:Sun601#**run**  
[xr-vm_node0_RP0_CPU0:~]$**sftp scadora@172.30.8.11**  
Connecting to 172.30.8.11...  
Password:  
  sftp> **get /tftpboot/BasicPolicy.policy /telemetry/policies/BasicPolicy.policy**  
  RP/0/RP0/CPU0:Jun 23 16:08:00.870 : sftp[69048]: %SECURITY-SSHD-3-ERR_GENERAL : Cannot overwrite system files  
  sftp>
 
The restriction on the /telemetry/policies directory will be lifted in 6.0.2, but in the meantime you can work around this by copying files to disk0: and then doing a local copy to the proper directory as follows:  
 
RP/0/RP0/CPU0:Sun601#**run**  
[xr-vm_node0_RP0_CPU0:~]$**sftp scadora@172.30.8.11**  
Connecting to 172.30.8.11...  
Password:  
sftp> **get /tftpboot/BasicPolicy.policy /disk0:/BasicPolicy.policy **  
  Transferred 469 Bytes  
  469 bytes copied in 0 sec (0)bytes/sec  
sftp> **quit**  
[xr-vm_node0_RP0_CPU0:~]$**cp /disk0:/BasicPolicy.policy /telemetry/policies**  
[xr-vm_node0_RP0_CPU0:~]$  

If you wish to copy a policy file from the host Linux operating system to the IOS XR filesystem within the Vagrant box, then there is a two-step process. Firstly, a writeable directory must be created in disk0:

RP/0/RP0/CPU0:ios#**run**
Tue Jun 28 06:40:16.346 UTC
[xr-vm_node0_RP0_CPU0:~]$**cd /disk0\:**
[xr-vm_node0_RP0_CPU0:~]$**mkdir mypolicies**
[xr-vm_node0_RP0_CPU0:~]$**chmod 777 mypolicies**

The policy file can then be copied using scp to the Vagrant IOS XR instance:

user@host:~/vagrant/iosxrv> **scp -P 2222 mypolicy.policy vagrant@localhost:/disk0:/mypolicies/**
vagrant@localhost's password: 
mypolicy.policy                                    100%  253     0.3KB/s   00:00 

Finally, the policy must be copied from its temporary location to the /telemetry/policies/ directory in the IOS XR filesystem:

[xr-vm_node0_RP0_CPU0:~]$**cp /disk0\:/mypolicies/cpu.policy /telemetry/policies/**

The policy is now loaded by IOS XR and is shown in the telemetry policy configuration:

RP/0/RP0/CPU0:ios#**show telemetry policies**
Tue Jun 28 07:36:39.125 UTC

cpu
  Filename:             cpu.policy
  Version:              25
  Description:          cpu
  Status:               Active
  [etc]

