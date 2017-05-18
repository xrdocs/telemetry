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

# gRPC Dialin
[dial-in](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/#grpc-dial-in)

### Without TLS

### With TLS
