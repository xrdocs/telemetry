---
published: false
date: '2017-05-08 11:03 -0600'
title: Pipeline with gRPC
author: Shelly Cadora
excerpt: Descibres how to use Model-Driven Telemetry with Pipeline and gRPC
tags:
  - iosxr
  - telemetry
  - gRPC
  - MDT
  - pipeline
---
# Using Pipeline with gRPC

In previous tutorials, I've shown how to use [Pipeline](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service) to dump Model Driven Telemetry (MDT) data into a [text file](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/) and into [InfluxDB](https://xrdocs.github.io/telemetry/tutorials/2017-04-10-using-pipeline-integrating-with-influxdb/).  In each case, I configured the router to transport MDT data to Pipeline using TCP.  In this tutorial, I'll cover a few additional steps that are required to use Pipeline with [gRPC](http://www.grpc.io/).  I'll focus on only the changes needed in the router and Pipeline input stage configs here, so be sure to consult the other Pipeline tutorialis for important info about install, output stage, etc.

If you're going to use gRPC, the first thing to decide is whether you're going to [dial-out](#dialout) or [dial-in](#dialin).  If you don't know the difference or need help chosing, check out [my blog](https://xrdocs.github.io/telemetry/blogs/2017-01-20-model-driven-telemetry-dial-in-or-dial-out/) for some guidance.        

# gRPC Dialout<a name=dialout></a>
For gRPC dial-out, the big decision is whether to use TLS or not. This impacts the destination-group in the router config and the ingress stage of the Pipeline input stage.

Aside the destination-group config, I'll re-use the rest of the MDT router config from the [gRPC dial-out example](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-out).  It should look something like this:

```
telemetry model-driven
 sensor-group SGroup2
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
 !
 subscription Sub2
  sensor-group-id SGroup2 sample-interval 30000
  destination-id DGroup2
``` 

## gRPC Dialout Without TLS
If you don't use TLS, your MDT data won't be encrypted and the server will have no way to authenticate the collector's identity before pushing data to it.  On the other hand, it's easy to configure. So if you're new to MDT and gRPC, this might be a good starting place.

### Router Config: no-tls

The gRPC config for the router is contained in the MDT destination-group.  Here is a destination-group for gRPC dialout without TLS:

```
telemetry model-driven
 destination-group DGroup2
  address family ipv4 172.30.8.4 port 57500
   encoding self-describing-gpb
   protocol grpc no-tls
```

### Pipeline.conf: tls = false

You can use the ```[gRPCDIalout]``` input stage in the default pipeline.conf.  Just uncomment the 6 lines shown below.

```
$ grep -A25 "gRPCDialout" pipeline.conf | grep -v -e '^#' -e '^$'
[gRPCDialout]
 stage = xport_input
 type = grpc
 encap = gpb
 listen = :57500
 tls = false
```

If you now run pipeline with the debug option, you should see these lines when Pipeline starts and the router (at 172.30.8.53) connects:
```
$ bin/pipeline -config pipeline.conf -log= -debug | grep gRPC
INFO[2017-05-08 11:25:50.046573] gRPC starting block                           encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-08 11:25:50.046902] gRPC: Start accepting dialout sessions        encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-08 11:26:03.572534] gRPC: Receiving dialout stream                encap=gpb name=grpcdialout peer="172.30.8.53:61857" server=:57500 tag=pipeline type="pipeline is SERVER"
```

## gRPC Dialout With TLS
In a dialout scenario, the router is the "client" in the gRPC connection and Pipeline is the "server."  Therefore, in the TLS handshake, Pipeline will need to send a certificate to authenticate itself to the router.  The router validates Pipeline's certificate using the public certificate of the Root Certificate Authority (CA) that signed it and then generates sesssion keys to encrypt the the session.

To make this all work, you need the following:
1. A Root CA certificate
2. A Pipeline certificate signed by the Root CA
3. A copy of the Root CA certificate on the router

For the purpose of this tutorial, I will use [openssl](https://www.openssl.org/) (an open-source TLS toolkit) for the root CA and Pipeline certificate.  If your organization has an existing PKI, you can skip the first two steps and just copy the Root CA certificate to the router.

### Creating Certificates

#### 1. The rootCA Key and Certificate
For simplicity, I'll generate the rootCA on the same server that I am running Pipeline.  First, create a rootCA key-pair (may require sudo):
```scadora@darcy:/etc/ssl/certs$ openssl genrsa -out rootCA.key 2048
Generating RSA private key, 2048 bit long modulus
...........+++
................................+++
e is 65537 (0x10001)
scadora@darcy:/etc/ssl/certs$
```

Now use that key to self-sign the rootCA certificate.  It will ask you a bunch of questions that you can fill out as you want (I just used all defaults):
```
scadora@darcy:/etc/ssl/certs$ sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -extensions v3_ca -config ../openssl.cnf -out rootCA.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:
scadora@darcy:/etc/ssl/certs$
```

You should now have a rootCA certificate called rootCA.pem.

#### 2. The Pipeline Certificate<a name=pipelinecert></a>
First, create a key pair for Pipeline.  In this case, I've called it "darcy.key" since darcy is the name of the server on which I am running Pipeline.
```scadora@darcy:/etc/ssl/certs$  sudo openssl genrsa -out darcy.key 2048
Generating RSA private key, 2048 bit long modulus
................+++
..+++
e is 65537 (0x10001)
scadora@darcy:/etc/ssl/certs$
```

Next, create a Certificate Signing Request (CSR) using the key you just generated.  In the following, I use all the defaults except for the Common Name, which I set as darcy.cisco.com:

```scadora@darcy:/etc/ssl/certs$ openssl req -new -key darcy.key -out darcy.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:darcy.cisco.example
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
scadora@darcy:/etc/ssl/certs$
```

Finally, use your rootCA certificate to sign the CSR ("darcy.csr") you just generated and create a certificate for Pipeline ("darcy.pem"):
```
cadora@darcy:/etc/ssl/certs$ openssl x509 -req -in darcy.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out darcy.pem -days 500 -sha256
Signature ok
subject=/C=AU/ST=Some-State/O=Internet Widgits Pty Ltd/CN=darcy.cisco.example
Getting CA Private Key
scadora@darcy:/etc/ssl/certs$
```

Note: Some people issue certificates with a Common Name set to the IP address of the server instead of a FQDN.  Should you do this for the Pipeline certificate, bear in mind that the certificate will also need to have a Subject Alternative Name section that explicitly lists all valid IP addresses.  If you see the following message in your grpc trace, this could be your problem.
```RP/0/RP0/CPU0:SunC#show grpc trace ems
Tue May 16 19:35:44.792 UTC
3 wrapping entries (141632 possible, 320 allocated, 0 filtered, 3 total)
May 16 19:35:40.240 ems/grpc 0/RP0/CPU0 t26842 EMS-GRPC: grpc: Conn.resetTransport failed to create client transport: connection error: desc = "transport: x509: cannot validate certificate for 172.30.8.4 because it doesn't contain any IP SANs"
```
{: .notice--warning}

#### 3. Copy rootCA Certificate to the router

For the router to validate Pipeline's certificate, it needs to have a copy of the rootCA certificate in /misc/config/grpc/dialout/dialout.pem (the filename is important!).  Here is how to scp the rootCA.pem to the appropriate file and directory:
```
RP/0/RP0/CPU0:SunC#bash
Tue May 16 18:06:04.592 UTC
[xr-vm_node0_RP0_CPU0:~]$ scp scadora@172.30.8.4://etc/ssl/certs/rootCA.pem /misc/config/grpc/dialout/dialout.pem
rootCA.pem                                    100% 1204     1.2KB/s   00:00
[xr-vm_node0_RP0_CPU0:~]$
```

### Configuring the Router
TLS is the default for MDT for gRPC, so the destination-group config just looks like this:

```
telemetry model-driven
 destination-group DGroup2
  address family ipv4 172.30.8.4 port 57500
   encoding self-describing-gpb
   protocol grpc 
```

### Configuring Pipeline for tls=true
We can use the ```[gRPCDIalout]``` input stage in the default pipeline.conf.  The only change is the last 4 lines where we enable tls and set the pem, key and servername to the values corresponding to the [Pipeline certificate we generated earlier](#pipelinecert). 

```
scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ grep -A30 "gRPCDialout" pipeline.conf | grep -v -e '^#' -e '^$'
[gRPCDialout]
 stage = xport_input
 type = grpc
 encap = gpb
 listen = :57500
 tls = true
 tls_pem = /etc/ssl/certs/darcy.pem
 tls_key = /etc/ssl/certs/darcy.key
 tls_servername = darcy.cisco.example
scadora@darcy:~/bigmuddy-network-telemetry-pipeline$
```

And that's it.  Run pipeline as usual and you'll see the router connect:
```scadora@darcy:~/bigmuddy-network-telemetry-pipeline$ sudo bin/pipeline -config pipeline.conf -log= -debug | grep gRPC
INFO[2017-05-16 13:00:40.976896] gRPC starting block                           encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-16 13:00:40.977505] gRPC: Start accepting dialout sessions        encap=gpb name=grpcdialout server=:57500 tag=pipeline type="pipeline is SERVER"
INFO[2017-05-16 13:01:09.775514] gRPC: Receiving dialout stream                encap=gpb name=grpcdialout peer="172.30.8.53:59865" server=:57500 tag=pipeline type="pipeline is SERVER"
```

On the router, the grpc trace will show you the server name (darcy.cisco.example) of the received certificate.
```
RP/0/RP0/CPU0:SunC#show grpc trace ems
Tue May 16 20:01:15.059 UTC
2 wrapping entries (141632 possible, 320 allocated, 0 filtered, 2 total)
May 16 20:01:06.757 ems/conf 0/RP0/CPU0 t26859 EMS-CONF:emsd_is_active_role get proc role (1)
May 16 20:01:06.759 ems/info 0/RP0/CPU0 t26843 EMS_INFO: nsDialerCheckAddTLSOption:322 mdtDialout: TLS pem: /misc/config/grpc/dialout/dialout.pem, Server host: darcy.cisco.example
RP/0/RP0/CPU0:SunC#
```

# gRPC Dialin<a name=dialin></a>

In a dialin scenario, Pipeline sends the SYN packet and acts as the "client" in the gRPC session and TLS handshake.

## Router Config for gRPC Dial-In<a name=router-dialin></a>
For this part of the tutorial, I'll re-use the MDT router config from the [gRPC dial-in example](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in)  It should look like this:

```
grpc
 port 57500
!
telemetry model-driven
 sensor-group SGroup3
  sensor-path openconfig-interfaces:interfaces/interface
 !
 subscription Sub3
  sensor-group-id SGroup3 sample-interval 30000
``` 

You will also need to configure a username and password that Pipeline can use when it dials in.  If you're just doing a quick test in the lab, assign the user to one of these default usergroups: sysadmin, netadmin, or root-lr.  

```
username mdt
 group sysadmin
 secret 5 $1$kAbv$xNk9KA.mIC7K2wfdpGjzk1
```

If you want to be more restrictive, here is the minimal taskgroup that you will need to support telemetry for gRPC dialin:

```
taskgroup mdt-grpc
 task read li
 task read acl
 task read cdp
 task read eem
 task read boot
 task read diag
 task read ipv4
 task read ipv6
 task read snmp
 task read vpdn
 task read crypto
 task read system
 task read logging
 task read fault-mgr
 task read interface
 task read ext-access
 task read filesystem
 task read tty-access
 task read config-mgmt
 task read ip-services
 task read host-services
 task read basic-services
 task read config-services
!
usergroup mdt-grpc
 taskgroup mdt-grpc
!
username mdt
 group mdt-grpc
 secret 5 $1$kAbv$xNk9KA.mIC7K2wfdpGjzk1
```

Next, you get to decide if you want to use TLS or not and forge ahead.

## gRPC Dialin Without TLS
If you don't use TLS, your MDT data won't be encrypted.  On the other hand, there's less fiddling with certificates. So if you're trying to get gRPC dialin to work for the first time, this might be a good starting place.

You can use the ```[mymdtrouter]``` input stage in the default pipeline.conf.  Just uncomment the 8 lines shown below, changing the server line to match your router's IP address and configured gRPC port:

```
$ grep -A48 "mymdtrouter" pipeline.conf | grep -v -e '^#' -e '^$'
 [mymdtrouter]
 stage = xport_input
 type = grpc
 encoding = gpbkv
 encap = gpb
 server = 172.30.8.53:57500
 subscriptions = Sub3
 tls = false
```

Note that the subscription is "Sub3", which matches the subscription in the router configuration [above](#router-dialin).  Pipeline will request the pre-configured subscription from the router when it connects.

When you run pipeline, you will be prompted for a username and password.  This is the username and password that you configured on the router [above](#router-dialin).

```
$ bin/pipeline -config pipeline.conf
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]

CRYPT Client [mymdtrouter],[172.30.8.53:57500]
 Enter username: mdt
 Enter password:
Wait for ^C to shutdown
```

Pipeline will connect to the router and the router will stream the telemetry data in Sub3 back to pipeline.  To verify that the connection is established, check that the subscription Destination Group State is Active:

```
RP/0/RP0/CPU0:SunC#show telemetry model sub Sub3
Thu May 18 20:46:38.658 UTC
Subscription:  Sub3
-------------
  State:       ACTIVE
  Sensor groups:
  Id: SGroup3
    Sample Interval:      30000 ms
    Sensor Path:          openconfig-interfaces:interfaces/interface
    Sensor Path State:    Resolved

  Destination Groups:
  Group Id: DialIn_1019
    Destination IP:       172.30.8.4
    Destination Port:     48667
    Encoding:             self-describing-gpb
    Transport:            dialin
    State:                Active
    No TLS
    Total bytes sent:     5723
    Total packets sent:   4
    Last Sent time:       2017-05-18 20:46:28.2143492698 +0000
```

## Secure password storage
Because Pipeline cares about your security, it won't let you store unencrypted router passwords in pipeline.conf.  If you dislike being prompted for a password every time you run it or you want to run pipeline in the background, you can have pipeline encrypt the password using the -pem option and store it in a new file as follows:

```
$ bin/pipeline -config pipeline.conf -pem ~/.ssh/id_rsa
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]

CRYPT Client [mymdtrouter],[172.30.8.53:57500]
 Enter username: mdt
 Enter password:
Generating sample config...A new configuration file [pipeline.conf_REWRITTEN] has been written including user name and encrypted password.
In future, you can run pipeline non-interactively.
Do remember to run pipeline with '-pem /home/scadora/.ssh/id_rsa -config pipeline.conf_REWRITTEN' options.
Wait for ^C to shutdown
```

If you take a look at the ```[mymdtrouter]``` stage in pipeline.conf_REWRITTEN, you'll see that the username and encrypted password are included:

```
$ grep -A10 "mymdtrouter" pipeline.conf_REWRITTEN | grep -v -e '^#' -e '^$'
[mymdtrouter]
subscriptions=Sub3
password=PZbS/IG4O+2lsok3xxBjQZwJ5CFyraixl//qdNy67IRMM1YMLlWqbbGHUXVGM1pX0HfKf7JU1beRivkOcwyANPff4hVmF5b7Ne1SBxnKS4VqSU+AMCN/e+FFHFrCA24m0ywTYB/Dt2PJZaUCQmYzxTwa71+Vxc7lHe2dtovH/DGutQfvRa2On6aHeqiQfMbBcEeKqwya4jtmexS11Dt1ai1QXqWgn2WiggvWTGcldANO4Nfkl4vICguVlrVEfNv16qNoPB/HerTNCuGLlBR0EBhxGPxCJteexAxadt68whG4UP/teTiD2qFZ2UFXCRnpnPvpic9LIZIaF4PgNg9AGw==
server=172.30.8.53:57500
type=grpc
encoding=gpbkv
encap=gpb
tls=false
username=mdt
stage=xport_input
```

Now you can run pipeline with the rewritten .conf file and it won't prompt you for the username again:

```
$ sudo bin/pipeline -config pipeline.conf_REWRITTEN -pem ~/.ssh/id_rsa
Startup pipeline
Load config from [pipeline.conf_REWRITTEN], logging in [pipeline.log]
Wait for ^C to shutdown
```

## gRPC Dialin With TLS
In a dialin scenario, the router is the "server" in the gRPC connection and Pipeline is the "client."  Therefore, in the TLS handshake, the router will need to send a certificate to authenticate itself to the router.  Pipeline validates the router's certificate using the public certificate of the Root Certificate Authority (CA) that signed it.