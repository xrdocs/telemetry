---
published: true
date: '2023-08-30 11:35 +0200'
title: 'Telemetry Stack Update: gRPC TLS'
author: Romain Cyrille
excerpt: gRPC dial-out or dial-in methods with TLS
tags:
  - iosxr
  - Telemetry
position: hidden
---
{% include toc icon="table" title="Table of Contents" %}

# Introduction 

For simplicity, Telemetry configuration is often shown without TLS. It is easier as no certificate management is required. However, a important benefit of gRPC is lost: encryption. While it can still be used, it is not recommended for production.  

This article is a follow up to my [previous article](https://xrdocs.io/telemetry/tutorials/telemetry-stack-update-qos-interface-statistics-example/) on the TIG stack. It will reuse the same configuration examples and describe the steps on how to use gRPC dial-out or dial-in methods with TLS for authentication and encryption between a Telemetry collector and IOS XR routers.

There was already a great article on using gRPC with TLS from Shelly [Pipeline with gRPC](https://xrdocs.io/telemetry/tutorials/2017-05-08-pipeline-with-grpc/). 
This article was much inspired by this previous one, though all configuration are now based on the Telegraf collector.
{: .notice}


# Certificates 

In order to use TLS, certificates are required. If you already have your own PKI (Public Key Infrastructure), you should be able to generate certificates for the Telegraf collector and routers

In the next section, we will show, as an example, how to create our own root CA and generates certificates to be used by gRPC.

While certificates can work with IP addresses only, many times it can cause issue. Therefore, FQDN (Fully qualified domain name) will be used. However, Subject Alternative Name can be used to explicitly define all valid IP addresses.
{: .notice--info}


## Private CA (Certificate Authority) 

Using the openssl command line, it is very easy to create our own CA. 

### Root CA certificate

The root certificate is the starting point of the trust chain. Once created, it can be deployed on devices to verify the authenticity of device certificates. It is also used to create those device certificates

1. The private key for the root CA must be created. A passphrase is requested for encrypting the key. It will be ask later when creating certificates
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    openssl genrsa -aes256 -out CA.key 2048
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl genrsa -aes256 -out CA.key 2048</span>
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ...........................................................+++++
    ............................................+++++
    e is 65537 (0x010001)
    Enter pass phrase for CA.key:
    Verifying - Enter pass phrase for CA.key:
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>
2. The root certificate is created with the private key. Here it will be valid for one year only. If used in production this may be changed.
Many question will be asked, you can fill the information as you prefer, this is only informational.
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color:yellow">openssl req -x509 -new -nodes -key CA.key -sha256 -days 365 -out CA.pem</span>
    Enter pass phrase for CA.key:
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:FR
    State or Province Name (full name) [Some-State]:Paris
    Locality Name (eg, city) []:
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Cisco
    Organizational Unit Name (eg, section) []:
    Common Name (e.g. server FQDN or YOUR name) []:lab
    Email Address []:
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>
3. The content of the new root certificate can be read with the following command. We can see that this certificate is a root certificate
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color:yellow">openssl x509 -text -noout -in CA.pem</span>
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number:
                65:8f:eb:27:12:78:df:4c:3f:e0:e3:c9:bb:80:38:55:f2:a1:29:6a
            Signature Algorithm: sha256WithRSAEncryption
            Issuer: C = FR, ST = Paris, O = Cisco, CN = lab
            Validity
                Not Before: Aug 17 16:14:00 2023 GMT
                Not After : Aug 16 16:14:00 2024 GMT
            Subject: C = FR, ST = Paris, O = Cisco, CN = lab
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    RSA Public-Key: (2048 bit)
                    Modulus:
                        00:ac:3f:17:3c:0b:60:cf:ec:e5:1f:b4:25:84:9b:
                        48:2e:29:54:f7:de:13:d9:5c:bd:c8:27:fc:c3:43:
                        43:b8:f9:c2:ae:25:52:3c:8e:87:a6:56:ad:4f:6d:
                        9b:b1:1a:38:c8:88:95:59:7c:5f:74:20:e7:cc:10:
                        2c:4e:8f:ee:33:0d:9d:b1:c9:bf:4a:8a:bc:a0:1b:
                        06:a7:8a:df:20:e4:ee:45:57:ae:6e:f1:01:a1:dd:
                        e9:16:f3:35:7b:a4:57:49:46:ac:25:8a:56:f2:17:
                        c1:fa:b2:40:6b:9e:d2:04:0e:79:11:fb:5f:ae:ed:
                        15:b5:8d:4b:6e:64:42:f6:cb:a3:4a:6a:83:a4:ae:
                        e8:da:b0:6e:a1:8e:2c:9d:47:16:fe:b1:5e:25:81:
                        9f:02:eb:b8:54:5a:7a:b8:da:3a:7d:b5:16:d1:91:
                        7c:89:2a:0c:8c:8d:60:42:7a:b5:dc:34:83:78:46:
                        6b:6c:35:b7:fc:35:97:25:52:a0:db:1c:88:c5:b9:
                        da:31:c7:99:8a:00:dc:0c:07:d4:9b:4a:45:ae:3d:
                        93:f7:c9:42:72:22:24:cf:3e:64:f0:12:49:91:cb:
                        07:c9:11:67:19:0c:a6:51:d8:c3:4e:d8:f8:29:2e:
                        b4:2b:f8:ff:b2:1c:f6:14:81:f7:e3:4f:8b:95:d6:
                        0e:b9
                    Exponent: 65537 (0x10001)
            X509v3 extensions:
                X509v3 Subject Key Identifier: 
                    50:5F:91:3E:C9:15:CF:23:CE:39:98:F4:70:73:40:CA:7D:D5:15:91
                X509v3 Authority Key Identifier: 
                    keyid:50:5F:91:3E:C9:15:CF:23:CE:39:98:F4:70:73:40:CA:7D:D5:15:91

                X509v3 Basic Constraints: critical
                    <span style="background-color: #7CFC00;">CA:TRUE</span>
        Signature Algorithm: sha256WithRSAEncryption
             96:52:3d:e6:d3:0e:7a:9d:af:ad:a7:0f:81:f3:cf:f9:4a:3f:
             9e:ed:14:94:bd:d6:73:5d:22:5f:1e:5e:68:f6:6e:bb:2a:4a:
             87:ee:2f:89:73:97:71:fe:71:19:8c:82:6c:49:f9:0e:e4:bd:
             e1:62:4a:d6:c3:2b:2f:6c:88:a3:a7:8c:3b:84:2f:d6:d5:9b:
             4b:49:4e:21:06:a5:3c:d1:db:d6:ab:6c:aa:1b:97:b7:14:32:
             04:9e:4d:75:7c:6c:3e:4e:bc:4c:6a:21:89:f8:80:f5:99:2c:
             d8:b4:7f:90:c7:8e:4b:ef:c2:6a:eb:fc:78:79:cf:2d:f1:91:
             9d:d3:af:b3:10:79:c6:d3:16:62:89:89:5c:90:e0:43:38:5a:
             7a:99:bd:81:68:73:6e:cc:1e:25:cb:5d:93:13:a5:a8:86:e6:
             c7:92:95:1b:e3:0b:6d:c5:0e:11:d0:91:93:fe:b3:f7:39:1e:
             75:9a:cd:98:8a:13:67:61:73:46:08:30:60:a4:29:3b:5a:ac:
             4c:0b:4f:18:27:ab:81:9d:32:20:d6:92:24:cb:70:2c:86:37:
             8e:44:60:06:e6:a1:fd:96:5b:d4:84:ce:9d:45:7e:26:3e:a3:
             06:e5:d3:ff:44:1a:3e:6f:9e:7a:7c:ad:5c:1d:9c:a1:68:c8:
             f3:1f:08:62
    cisco@server1:/home/cisco#
    </code>
    </pre>
    </div>

### Telegraf certificate

Once the root certificate is created. Many devices certificates can be created. One certificate is created for the telegraf collector.

1. First, the private key for the Telegraf certificate must be created. A passphrase is requested for encrypting the key
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color:yellow">openssl genrsa -aes256 -out telegraf.lab.key 2048</span>
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ............................................................................................................................+++++
    ...............+++++
    e is 65537 (0x010001)
    Enter pass phrase for telegraf.lab.key:
    Verifying - Enter pass phrase for telegraf.lab.key:
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>

2. A certificate signing request (CSR) needs to be created with the private key in order to request the certificate to the CA. A configuration file is used to generate the CSR. In the example below, IP addresses are defined as SAN (Subject Alternative Name)
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# cat telegraf.lab.conf 
    distinguished_name = req_distinguished_name
    req_extensions = req_ext
    prompt = no

    [req_ext]
    subjectAltName = @alt_names

    [req_distinguished_name]
    C   = FR
    ST  = Paris
    L   = Paris
    O   = Cisco
    CN  = telegraf.lab

    [server_cert]
    subjectAltName = @alt_names
    basicConstraints = CA:FALSE
    nsCertType = server
    nsComment = "OpenSSL Generated Server Certificate"
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer:always
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth

    [alt_names]
    IP.1 = 192.168.122.1
    IP.2 = 10.48.82.175
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl req -new -key telegraf.lab.key -out telegraf.lab.csr -config telegraf.lab.conf</span>
    Enter pass phrase for telegraf.lab.key:
    cisco@server1:/home/cisco#
    </code>
    </pre>
    </div>
3. The created CSR can be read with the following command:
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl req -text -noout -in telegraf.lab.csr</span>
    Certificate Request:
        Data:
            Version: 1 (0x0)
            Subject: C = FR, ST = Paris, L = Paris, O = Cisco, CN = telegraf.lab
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    RSA Public-Key: (2048 bit)
                    Modulus:
                        00:f6:6e:0a:a0:3c:50:b2:69:4b:9b:89:86:ba:62:
                        77:5f:12:7f:58:e8:1e:bb:d7:a3:da:c1:d5:75:72:
                        e7:04:f3:7b:b6:03:88:ea:be:a3:3b:da:30:07:7d:
                        55:1a:92:32:8a:88:e9:d2:be:6a:56:7f:9d:2f:39:
                        2f:31:8a:4e:85:7a:3a:34:fe:5c:87:28:0f:65:4b:
                        6a:36:ab:d0:3c:f5:25:28:be:46:13:cc:a5:e9:ba:
                        fb:4b:e1:7b:fe:41:eb:c5:4e:fa:2d:d0:4d:c9:cf:
                        c3:9b:f6:3e:8e:bc:9b:fd:6d:9e:b3:f8:f1:f7:fb:
                        e8:f8:21:89:5d:84:f8:2e:b5:89:d9:8d:1c:0d:91:
                        9d:b6:db:3b:dd:db:46:7d:02:98:0f:b4:05:94:95:
                        49:07:b5:bf:dd:d7:82:44:ff:64:d9:75:17:55:c9:
                        71:e9:1b:9a:2f:37:b6:14:a2:af:68:e2:cc:4a:f4:
                        4a:f0:d1:83:38:f3:c1:fa:eb:7c:de:42:f4:4f:4d:
                        c1:6b:65:a2:e5:5b:0c:e1:42:ce:34:19:3f:f0:d9:
                        f7:a7:2b:63:85:db:90:84:dd:92:02:06:51:d0:46:
                        b4:0b:d2:46:b9:e9:b3:90:b9:5a:4b:fa:6b:13:31:
                        c2:38:c7:7b:0a:be:55:59:3b:2e:1a:82:49:07:89:
                        65:e9
                    Exponent: 65537 (0x10001)
            Attributes:
            Requested Extensions:
                X509v3 Subject Alternative Name: 
                    IP Address:192.168.122.1, IP Address:10.48.82.175
        Signature Algorithm: sha256WithRSAEncryption
             3b:8c:0b:0c:d8:9d:29:a7:6a:bc:44:25:26:9f:92:cd:fa:43:
             3a:c5:0d:b2:bd:ff:58:4e:ec:c6:b4:17:d1:c6:0a:b6:d7:3c:
             b0:d4:34:4e:52:ca:9c:48:4a:50:0b:ea:bc:c3:26:e8:cc:bc:
             3f:68:9b:cd:e9:5c:34:1f:46:40:f5:05:ce:3f:26:e3:51:48:
             5c:81:b5:6a:91:19:f4:29:61:c9:12:02:5a:80:4a:e0:b5:ea:
             f6:2a:a8:b3:7b:a1:91:77:af:5b:bf:15:6a:50:12:74:b0:a0:
             3a:57:04:02:48:6a:0b:d6:75:56:ee:7c:df:e1:85:c7:fd:b6:
             87:30:28:ce:36:1f:ea:5e:44:62:76:61:76:da:b3:ec:9f:5e:
             a6:c4:68:af:61:27:0f:e0:09:d6:72:d7:bd:d8:28:ae:87:01:
             02:86:a6:45:77:0e:2e:9b:d6:ae:64:b4:a6:93:f9:14:e7:d3:
             cd:b9:0b:e8:7d:91:ee:54:c6:36:ad:5e:08:46:39:c9:ef:c7:
             fe:be:aa:0c:52:08:fe:5e:c7:d0:86:3c:58:44:7b:92:d1:57:
             56:8f:fe:d2:46:8d:1b:a2:ba:34:a2:f2:0d:9a:85:be:5b:c2:
             17:7b:14:8b:93:7b:94:3f:f7:41:45:9f:86:c3:2d:64:a2:5c:
             83:cf:89:6c
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>
4. From the CSR, the device certificate can be created with the root CA certificate and the CA key. The same config file is  provided.
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl x509 -req -in telegraf.lab.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out telegraf.lab.pem -days 180 -sha256 -extfile telegraf.lab.conf -extensions server_cert</span>
    Signature ok
    subject=C = FR, ST = Paris, L = Paris, O = Cisco, CN = telegraf.lab
    Getting CA Private Key
    Enter pass phrase for CA.key:
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>
5. Finally, we can verify the content of the created certificate:
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco#  <span style="background-color: yellow">openssl x509 -text -noout -in telegraf.lab.pem</span>
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number:
                2a:06:27:60:ac:cb:cc:06:88:c8:60:94:d4:11:55:e0:86:0f:10:30
            Signature Algorithm: sha256WithRSAEncryption
            Issuer: C = FR, ST = Paris, O = Cisco, CN = lab
            Validity
                Not Before: Aug 29 15:54:11 2023 GMT
                Not After : Feb 25 15:54:11 2024 GMT
            Subject: C = FR, ST = Paris, L = Paris, O = Cisco, CN = telegraf.lab
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    RSA Public-Key: (2048 bit)
                    Modulus:
                        00:f6:6e:0a:a0:3c:50:b2:69:4b:9b:89:86:ba:62:
                        77:5f:12:7f:58:e8:1e:bb:d7:a3:da:c1:d5:75:72:
                        e7:04:f3:7b:b6:03:88:ea:be:a3:3b:da:30:07:7d:
                        55:1a:92:32:8a:88:e9:d2:be:6a:56:7f:9d:2f:39:
                        2f:31:8a:4e:85:7a:3a:34:fe:5c:87:28:0f:65:4b:
                        6a:36:ab:d0:3c:f5:25:28:be:46:13:cc:a5:e9:ba:
                        fb:4b:e1:7b:fe:41:eb:c5:4e:fa:2d:d0:4d:c9:cf:
                        c3:9b:f6:3e:8e:bc:9b:fd:6d:9e:b3:f8:f1:f7:fb:
                        e8:f8:21:89:5d:84:f8:2e:b5:89:d9:8d:1c:0d:91:
                        9d:b6:db:3b:dd:db:46:7d:02:98:0f:b4:05:94:95:
                        49:07:b5:bf:dd:d7:82:44:ff:64:d9:75:17:55:c9:
                        71:e9:1b:9a:2f:37:b6:14:a2:af:68:e2:cc:4a:f4:
                        4a:f0:d1:83:38:f3:c1:fa:eb:7c:de:42:f4:4f:4d:
                        c1:6b:65:a2:e5:5b:0c:e1:42:ce:34:19:3f:f0:d9:
                        f7:a7:2b:63:85:db:90:84:dd:92:02:06:51:d0:46:
                        b4:0b:d2:46:b9:e9:b3:90:b9:5a:4b:fa:6b:13:31:
                        c2:38:c7:7b:0a:be:55:59:3b:2e:1a:82:49:07:89:
                        65:e9
                    Exponent: 65537 (0x10001)
            X509v3 extensions:
                X509v3 Subject Alternative Name: 
                    <span style="background-color: #7CFC00;">IP Address:192.168.122.1, IP Address:10.48.82.175</span>
                X509v3 Basic Constraints: 
                    <span style="background-color: #7CFC00;">CA:FALSE</span>
                Netscape Cert Type: 
                    SSL Server
                Netscape Comment: 
                    OpenSSL Generated Server Certificate
                X509v3 Subject Key Identifier: 
                    C2:8A:9B:43:7C:7D:0F:D0:9A:00:69:20:E9:91:80:9D:A0:05:BA:66
                X509v3 Authority Key Identifier: 
                    keyid:50:5F:91:3E:C9:15:CF:23:CE:39:98:F4:70:73:40:CA:7D:D5:15:91
                    DirName:/C=FR/ST=Paris/O=Cisco/CN=lab
                    serial:65:8F:EB:27:12:78:DF:4C:3F:E0:E3:C9:BB:80:38:55:F2:A1:29:6A

                X509v3 Key Usage: critical
                    Digital Signature, Key Encipherment
                X509v3 Extended Key Usage: 
                    TLS Web Server Authentication
        Signature Algorithm: sha256WithRSAEncryption
             1c:20:6e:81:24:70:97:59:bd:89:12:c8:1c:9a:11:1f:0f:88:
             91:84:5a:d9:d9:72:b2:84:87:11:0c:e2:5b:42:d9:6f:26:74:
             05:86:74:e6:36:59:f9:fd:2f:c6:b5:c1:c6:6d:17:9b:8c:73:
             d1:4e:35:a5:d4:44:5d:dd:f7:02:5b:c3:08:38:f2:d3:0a:df:
             b3:c8:0c:79:d9:94:4d:d6:53:ac:9f:a4:8d:bb:05:00:f8:87:
             1b:5a:ce:7d:d0:f9:8c:20:92:dc:6a:f5:d9:8f:2d:6d:55:12:
             24:40:2d:73:b1:2c:b6:31:8b:a6:a9:67:b6:1d:f9:9b:dc:31:
             ef:72:81:74:e0:a6:00:5e:95:15:38:f7:35:7d:ab:82:85:e9:
             11:55:9c:df:a6:77:56:2d:45:c9:1d:57:bc:a7:78:e9:76:e6:
             2f:7c:26:f6:83:33:78:f4:a1:d9:06:5c:b3:f1:eb:bb:ab:3d:
             00:80:f9:46:e6:71:ec:9a:9c:0a:bb:0c:0f:f0:77:b0:10:64:
             d4:ff:43:bf:52:1d:6f:b9:5e:f4:e8:4f:fa:c1:b2:7a:5e:97:
             0c:a6:e9:9f:7d:b7:2a:be:8e:6b:a3:21:6b:72:bb:d7:ed:15:
             93:4d:97:b8:38:c5:3f:fa:61:cc:77:05:fe:9e:9f:bd:7c:c9:
             75:5f:1b:a9
    cisco@server1:/home/cisco#
    </code>
    </pre>
    </div>

### Routers certificates

The same procedure as for the Telegraf certificate can be applied for the routers certificates. To ease generation and deployment, a wildcard certificate will be used. It allows a single certificate to be used on multiples devices using a single wildcard character (\*). Doing this as pro and cons, as I am not a security expert I will not comment whether this should be used in production. 

When using the dial-out method, no router certificate is required. When using the dial-in method, routers are seen as telemetry servers as the collector initiate the TLS session, therefore certificates are required on the routers. The Telegraf certificate may also be required for the dial-in method when mutual authentication is used. It provides authentication of the collector connecting to the routers. It prevents a potential rogue collector that would be able to collect data on the routers.
{: .notice--info}

1. First, the private key for the routers certificate must be created. A passphrase is requested for encrypting the key
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color:yellow">openssl genrsa -aes256 -out routers.lab.key 2048</span>
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ........................+++++
    .............................+++++
    e is 65537 (0x010001)
    Enter pass phrase for routers.lab.key:
    Verifying - Enter pass phrase for routers.lab.key:
    root@vxr8000:/home/cisco/telemetry_docker#
    </code>
    </pre>
    </div>

2. A certificate signing request (CSR) needs to be created with the private key in order to request the certificate to the CA. A configuration file is used to generate the CSR, notice that a wildcard (\*) is used in the common name (CN) field.
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    root@vxr8000:/home/cisco/telemetry_docker# cat routers.lab.conf 
    distinguished_name = req_distinguished_name
    prompt = no

    [req_distinguished_name]
    C   = FR
    ST  = Paris
    L   = Paris
    O   = Cisco
    <span style="background-color:grey">CN  = \*.routers.lab</span>

    [server_cert]
    basicConstraints = CA:FALSE
    nsCertType = server
    nsComment = "OpenSSL Generated Server Certificate"
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer:always
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth
    root@vxr8000:/home/cisco/telemetry_docker# <span style="background-color:yellow">openssl req -new -key routers.lab.key -out routers.lab.csr -config routers.lab.conf</span>
    Enter pass phrase for routers.lab.key:
    root@vxr8000:/home/cisco/telemetry_docker# 
    </code>
    </pre>
    </div>
3. From the CSR, the device certificate can be created with the root CA certificate and the CA key. The same config file is  provided.
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl x509 -req -in routers.lab.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out routers.lab.pem -days 180 -sha256 -extfile routers.lab.conf -extensions server_cert</span>
    Signature ok
    subject=C = FR, ST = Paris, L = Paris, O = Cisco, CN = *.routers.lab
    Getting CA Private Key
    Enter pass phrase for CA.key:
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>
4. Finally, we can verify the content of the created certificate, the wildcard can be verified in the CN field.
    <div class="highlighter-rouge">
    <pre class="highlight">
    <code>
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl x509 -text -noout -in routers.lab.pem</span>
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number:
                2a:06:27:60:ac:cb:cc:06:88:c8:60:94:d4:11:55:e0:86:0f:10:31
            Signature Algorithm: sha256WithRSAEncryption
            Issuer: C = FR, ST = Paris, O = Cisco, CN = lab
            Validity
                Not Before: Aug 30 09:30:02 2023 GMT
                Not After : Feb 26 09:30:02 2024 GMT
            Subject: C = FR, ST = Paris, L = Paris, O = Cisco, <span style="background-color: #7CFC00;">CN = *.routers.lab</span>
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    RSA Public-Key: (2048 bit)
                    Modulus:
                        00:b1:a4:e7:82:a5:2f:ca:91:71:2e:36:ac:35:64:
                        e7:0f:92:6d:98:2d:17:0c:49:22:4c:0d:57:a6:a8:
                        89:81:18:cd:20:65:35:31:6c:c4:a0:d4:8f:48:65:
                        71:26:30:83:c8:d6:47:47:f4:3c:73:06:37:11:fe:
                        97:0e:37:55:5b:a5:7a:0a:7b:33:8d:b7:03:fe:63:
                        f0:f1:2f:2a:88:18:b6:f7:5f:a6:09:b6:e3:1e:59:
                        2e:2a:0b:d1:ce:14:33:15:4c:47:9e:3e:3b:ef:5e:
                        07:c6:93:a2:22:31:c1:e1:20:df:2c:64:09:df:e5:
                        fe:31:d1:b4:d0:e6:17:de:e6:9c:dd:af:70:5e:38:
                        12:49:76:ac:b7:32:8e:de:96:a2:b5:a7:9c:1f:98:
                        31:b9:3d:a6:8a:df:a2:b9:96:8d:a0:f5:c7:6c:5d:
                        18:6d:06:51:48:ce:d1:a2:b2:88:af:97:ae:9c:48:
                        2a:58:c4:3c:3d:d8:2e:e9:cd:9f:0b:ab:3b:df:11:
                        1f:dc:33:69:36:09:f2:9c:3f:aa:45:5b:bb:5c:5c:
                        fe:b7:ed:59:d8:3d:32:b4:89:7a:aa:49:e8:5b:0f:
                        cb:ac:bf:b3:74:86:6b:64:af:eb:68:1b:60:09:f1:
                        4a:7b:7e:66:ec:0f:3d:84:74:3e:57:3e:1d:d2:21:
                        3d:83
                    Exponent: 65537 (0x10001)
            X509v3 extensions:
                X509v3 Basic Constraints: 
                    <span style="background-color: #7CFC00;">CA:FALSE</span>
                Netscape Cert Type: 
                    SSL Server
                Netscape Comment: 
                    OpenSSL Generated Server Certificate
                X509v3 Subject Key Identifier: 
                    50:E9:E9:15:A5:80:A5:72:B2:FD:40:5D:6A:0F:80:72:BC:42:DE:3A
                X509v3 Authority Key Identifier: 
                    keyid:50:5F:91:3E:C9:15:CF:23:CE:39:98:F4:70:73:40:CA:7D:D5:15:91
                    DirName:/C=FR/ST=Paris/O=Cisco/CN=lab
                    serial:65:8F:EB:27:12:78:DF:4C:3F:E0:E3:C9:BB:80:38:55:F2:A1:29:6A

                X509v3 Key Usage: critical
                    Digital Signature, Key Encipherment
                X509v3 Extended Key Usage: 
                    TLS Web Server Authentication
        Signature Algorithm: sha256WithRSAEncryption
             4e:3d:1c:80:ab:85:c4:8b:9a:a6:ed:ec:34:1a:87:1e:8e:4d:
             fe:f6:76:bd:39:a2:cd:cf:41:11:e8:91:99:d2:4f:ca:fe:66:
             e1:d6:38:c8:8e:3a:88:7e:f2:be:50:ca:9c:ce:0e:66:94:1e:
             09:cc:57:57:b2:4e:df:a3:2e:bb:21:1e:f0:93:ea:b4:dc:3a:
             d8:49:b7:b5:a6:4c:86:b4:e0:1c:4d:9d:43:2d:84:c4:1a:30:
             76:67:f1:e3:6b:30:c6:8f:62:a9:da:88:59:87:8e:84:83:a8:
             b3:0e:57:33:ef:77:13:07:28:ce:f9:30:34:a5:65:74:c8:0d:
             e9:3f:70:a4:68:00:f0:2c:0c:dd:e0:54:46:46:7e:a0:92:2b:
             99:6f:39:a3:be:cc:14:a3:07:55:5b:df:a9:1c:96:cc:49:75:
             d3:ce:e3:93:40:20:ff:6d:6d:90:ff:5f:85:35:57:c8:e3:82:
             93:d3:2b:0e:b4:a4:79:86:6a:0a:e3:89:d1:fc:96:58:2c:17:
             02:b5:d3:62:56:43:d8:f4:5a:d9:b2:c1:78:b3:19:f9:ae:b1:
             9b:00:e0:f5:79:e5:0f:67:db:26:47:a0:29:5a:33:24:f6:94:
             19:20:d0:20:2e:f0:58:6d:c1:6f:e9:07:a1:03:a4:9c:f2:0d:
             e8:23:be:db
    cisco@server1:/home/cisco# 
    </code>
    </pre>
    </div>

## Required Certificates 
This section summarizes what certificate files are required for the dial-out and the dial-in methods

### Dial-out method

For the dial-out methods, the following three files are required:
 - **CA.pem:** the root CA certificate. It will be copied on the routers to be able to verify the Telegraf certificate
 - **telegraf.lab.pem:** the Telegraf certificate that will be used as the server certificate for the Telegraf collector
 - **telegraf.lab.key:** the private key associated with the Telegraf certificate

### Dial-in method
For the dial-in methods, the following three files are required:
 - **CA.pem:** the root CA certificate. It will be provided to the Telegraf collector to verify the routers certificates
 - **routers.lab.pem:** the routers certificate that will be used as the server certificate for the routers telemetry servers
 - **routers.lab.key:** the private key associated with the routers certificate

Optionally, if mutual authentication is used, the following files are also required on the Telegraf collector.
 - **telegraf.lab.pem:** the Telegraf certificate that will be used as the server certificate for the Telegraf collector
 - **telegraf.lab.key:** the private key associated with the Telegraf certificate
