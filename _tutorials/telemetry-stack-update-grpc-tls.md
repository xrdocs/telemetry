---
published: true
date: '2023-09-11 11:35 +0200'
title: 'Telemetry Stack Update: gRPC TLS'
author: Romain Cyrille
excerpt: gRPC dial-out or dial-in methods with TLS
tags:
  - iosxr
  - Telemetry
position: top
---
{% include toc icon="table" title="Table of Contents" %}

# Introduction 

For simplicity, Telemetry configuration is often shown without TLS. It is easier as no certificate management is required. However, an important benefit of gRPC is lost: encryption. While it can still be used, it is not recommended for production.  

This article is a follow up to my [previous article](https://xrdocs.io/telemetry/tutorials/telemetry-stack-update-qos-interface-statistics-example/) on the TIG stack. It will reuse the same configuration examples and describe the steps on how to use gRPC dial-out or dial-in methods with TLS, for authentication and encryption between a Telemetry collector and IOS XR routers. All configuration files presented can be found on the [Github repository](https://github.com/RomainCyr/tig-stack-qos-interface-statistics/tree/main/tls)

There was already a great article on using gRPC with TLS from Shelly [Pipeline with gRPC](https://xrdocs.io/telemetry/tutorials/2017-05-08-pipeline-with-grpc/). 
This article was much inspired by this previous one, though all configurations are now based on the Telegraf collector.


# Certificates 

In order to use TLS, certificates are required. If you already have your own PKI (Public Key Infrastructure), you should be able to generate certificates for the Telegraf collector and routers.

In the next section, we will show, as an example, how to create our own root CA and generates certificates to be used by gRPC.

While certificates can work with IP addresses only, many times it can cause issue. Therefore, FQDN (Fully qualified domain name) will be used. However, Subject Alternative Name can be used to explicitly define all valid IP addresses.
{: .notice--info}


## Private CA (Certificate Authority) 

Using the openssl command line, it is very easy to create our own CA. 

### Root CA certificate

The root certificate is the starting point of the trust chain. Once created, it can be deployed on devices to verify the authenticity of device certificates. It is also used to create those device certificates.

1. The private key for the root CA must be created. A passphrase is requested for encrypting the key. It will be asked later when creating certificates
    <div class="highlighter-rouge"><pre class="highlight"><code>openssl genrsa -aes256 -out CA.key 2048
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl genrsa -aes256 -out CA.key 2048</span>
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ...........................................................+++++
    ............................................+++++
    e is 65537 (0x010001)
    Enter pass phrase for CA.key:
    Verifying - Enter pass phrase for CA.key:
    cisco@server1:/home/cisco# 
    </code></pre></div>
2. The root certificate is created with the private key. Here it will be valid for one year only. If used in production this may be changed.
Many questions will be asked, you can fill the information as you prefer, this is only informational.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color:yellow">openssl req -x509 -new -nodes -key CA.key -sha256 -days 365 -out CA.pem</span>
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
    </code></pre></div>
3. The content of the new root certificate can be read with the following command. We can see that this certificate is a root certificate
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color:yellow">openssl x509 -text -noout -in CA.pem</span>
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
    </code></pre></div>

### Telegraf certificate

Once the root certificate is created. Many devices certificates can be created. One certificate is created for the Telegraf collector.

1. First, the private key for the Telegraf certificate must be created.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color:yellow">openssl genrsa -out telegraf.lab.key 2048</span>
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ............................................................................................................................+++++
    ...............+++++
    e is 65537 (0x010001)
    cisco@server1:/home/cisco# 
    </code></pre></div>

2. A certificate signing request (CSR) needs to be created with the private key in order to request the certificate to the CA. A configuration file is used to generate the CSR. In the example below, IP addresses are also defined in the SAN (Subject Alternative Name).
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# cat telegraf.lab.conf 
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
    <span style="background-color: #7CFC00;">CN  = telegraf.lab</span>

    [server_cert]
    subjectAltName = @alt_names
    basicConstraints = CA:FALSE
    nsComment = "OpenSSL Generated Server Certificate"
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer:always
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth

    [alt_names]
    <span style="background-color: #7CFC00;">DNS.1 = telegraf.lab</span>
    IP.1 = 192.168.122.1
    IP.2 = 10.48.82.175
    cisco@server1:/home/cisco# <span style="background-color: yellow">openssl req -new -key telegraf.lab.key -out telegraf.lab.csr -config telegraf.lab.conf</span>
    cisco@server1:/home/cisco#
    </code></pre></div>
3. The created CSR can be read with the following command:
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color: yellow">openssl req -text -noout -in telegraf.lab.csr</span>
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
                    DNS:telegraf.lab, IP Address:192.168.122.1, IP Address:10.48.82.175 
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
    </code></pre></div>
4. From the CSR, the device certificate can be created with the root CA certificate and the CA key. The same config file is  provided.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color: yellow">openssl x509 -req -in telegraf.lab.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out telegraf.lab.pem -days 180 -sha256 -extfile telegraf.lab.conf -extensions server_cert</span>
    Signature ok
    subject=C = FR, ST = Paris, L = Paris, O = Cisco, CN = telegraf.lab
    Getting CA Private Key
    Enter pass phrase for CA.key:
    cisco@server1:/home/cisco# 
    </code></pre></div>
5. Finally, we can verify the content of the created certificate:
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco#  <span style="background-color: yellow">openssl x509 -text -noout -in telegraf.lab.pem</span>
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
                    <span style="background-color: #7CFC00;">DNS:telegraf.lab, IP Address:192.168.122.1, IP Address:10.48.82.175</span>
                X509v3 Basic Constraints: 
                    <span style="background-color: #7CFC00;">CA:FALSE</span>
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
                    TLS Web Server Authentication, TLS Web Client Authentication
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
    </code></pre></div>

### Routers certificates

The same procedure as for the Telegraf certificate can be applied for the routers certificates. To ease generation and deployment, a wildcard certificate will be used. It allows a single certificate to be used on multiples devices using a wildcard character (\*). Doing this as pro and cons, and, as I am not a security expert, I will not comment whether this should be used in production. 

When using the dial-out method, no router certificate is required. When using the dial-in method, routers are seen as TLS servers as the collector initiate the telemetry session, therefore certificates are required on the routers. The Telegraf certificate may also be required for the dial-in method when mutual authentication is used. It provides authentication of the collector connecting to the routers. It prevents a potential rogue collector that would be able to collect data on the routers.
{: .notice--info}

1. First, the private key for the routers certificate must be created.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color:yellow">openssl genrsa -out routers.lab.key 2048</span>
    Generating RSA private key, 2048 bit long modulus (2 primes)
    ........................+++++
    .............................+++++
    e is 65537 (0x010001)
    cisco@server1:/home/cisco#
    </code></pre></div>

2. A certificate signing request (CSR) needs to be created with the private key in order to request the certificate to the CA. A configuration file is used to generate the CSR, notice that a wildcard (\*) is used for the subject name of the certificate.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# cat routers.lab.conf 
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
    <span style="background-color: #7CFC00;">CN  = \*.routers.lab</span>

    [server_cert]
    subjectAltName = @alt_names
    basicConstraints = CA:FALSE
    nsComment = "OpenSSL Generated Server Certificate"
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid,issuer:always
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth

    [alt_names]
    <span style="background-color: #7CFC00;">DNS.1 = \*.routers.lab</span>
    cisco@server1:/home/cisco# <span style="background-color:yellow">openssl req -new -key routers.lab.key -out routers.lab.csr -config routers.lab.conf</span>
    cisco@server1:/home/cisco# 
    </code></pre></div>
3. From the CSR, the device certificate can be created with the root CA certificate and the CA key. The same config file is  provided.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color: yellow">openssl x509 -req -in routers.lab.csr -CA CA.pem -CAkey CA.key -CAcreateserial -out routers.lab.pem -days 180 -sha256 -extfile routers.lab.conf -extensions server_cert</span>
    Signature ok
    subject=C = FR, ST = Paris, L = Paris, O = Cisco, CN = *.routers.lab
    Getting CA Private Key
    Enter pass phrase for CA.key:
    cisco@server1:/home/cisco# 
    </code></pre></div>
4. Finally, we can verify the content of the created certificate with the wildcard subject name:
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco# <span style="background-color: yellow">openssl x509 -text -noout -in routers.lab.pem</span>
    Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            06:a4:47:be:8e:9a:38:7e:57:a3:ff:e8:3f:4f:f1:19:98:91:50:37
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = FR, ST = Paris, O = Cisco, CN = lab
        Validity
            Not Before: Aug 31 14:18:42 2023 GMT
            Not After : Feb 27 14:18:42 2024 GMT
        Subject: C = FR, ST = Paris, L = Paris, O = Cisco, CN = <span style="background-color: #7CFC00;">*.routers.lab</span>
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
            X509v3 Subject Alternative Name: 
                DNS:<span style="background-color: #7CFC00;">*.routers.lab</span>
            X509v3 Basic Constraints: 
                <span style="background-color: #7CFC00;">CA:FALSE</span>
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
                TLS Web Server Authentication, TLS Web Client Authentication
    Signature Algorithm: sha256WithRSAEncryption
         7b:a4:f3:84:d2:d1:4e:38:e5:85:af:f7:32:f1:86:f7:d3:68:
         d6:c5:0b:9a:75:89:67:7d:b3:a1:83:bf:6f:f3:df:1c:ad:1a:
         52:bf:be:7d:48:2a:c4:27:6f:19:cc:01:ee:35:3e:40:aa:2e:
         98:70:08:bb:9f:cd:ed:3a:ed:49:b5:c8:30:55:4e:ac:87:2e:
         4d:f3:b5:e4:b7:47:4f:29:09:5f:ca:db:64:60:56:07:77:6a:
         aa:7e:42:63:53:32:30:03:30:58:33:ed:81:b4:74:9c:2f:3e:
         24:39:47:91:4d:af:b8:aa:8e:64:9e:cf:f6:77:43:7c:d7:54:
         6e:4d:10:b3:26:fe:95:60:6b:81:ee:91:85:77:5e:2a:b1:53:
         af:1f:56:19:de:5e:b3:c2:6a:8c:f3:e1:57:f8:b4:a5:f3:05:
         1d:40:57:09:1c:b1:d1:e8:4c:3e:54:9a:8a:b8:64:a5:a5:7f:
         c9:4a:14:55:b4:9a:ae:b5:7b:64:40:30:80:dc:79:06:4d:68:
         2a:8a:c1:71:64:08:b0:2c:be:62:3e:3f:88:e5:c3:fe:5e:37:
         96:ae:8f:00:7f:a9:4d:1c:39:28:14:4f:dc:ed:9d:2e:6c:39:
         6e:36:64:6b:7b:36:b5:05:87:3f:69:39:e5:1b:87:f8:85:b3:
         7b:82:cc:2e
    cisco@server1:/home/cisco# 
    </code></pre></div>

## Required Certificates 
This section summarizes what certificate files are required for the dial-out and the dial-in methods.
Those files are required, whether or not you have created your own certificates using the examples above.

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

# Docker compose
The same docker compose for the TIG stack is reused. Only the root CA certificate and the Telegraf certificate and private key are added to the Telegraf container. Note that if the method used is dial-in with no mutual-authentication, then only the root CA certificate is required.
<div class="highlighter-rouge">
<pre class="highlight">
<code>version: "2"
services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - '3000:3000'
    volumes:
      - ./grafana-provisioning/:/etc/grafana/provisioning
    depends_on:
      - influxdb
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - INFLUX_DB_TOKEN=MYSUPERSECRETTOKEN

  influxdb:
    image: influxdb:latest
    container_name: influxdb
    ports:
      - '8086:8086'
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_BUCKET=telemetry
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=admin123
      - DOCKER_INFLUXDB_INIT_ORG=lab
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=MYSUPERSECRETTOKEN
    volumes: 
      - /tmp/influxdb2_data:/var/lib/influxdb2

  telegraf:
    image: telegraf:latest
    container_name: telegraf
    ports:
      - '57500:57500'
    volumes:
     # - ./telegraf_dial_out.conf:/etc/telegraf/telegraf.conf:ro
     - ./telegraf_dial_in.conf:/etc/telegraf/telegraf.conf:ro
     - ./embedded_tag.star:/etc/telegraf/embedded_tag.star:ro
     <span style="background-color:#F0FFFF;">- ./pki/telegraf.lab.key:/etc/telegraf/key.pem:ro</span>
     <span style="background-color:#F0FFFF;">- ./pki/telegraf.lab.pem:/etc/telegraf/cert.pem:ro</span>
     <span style="background-color:#F0FFFF;">- ./pki/CA.pem:/etc/telegraf/CA.pem:ro</span>
</code>
</pre>
</div>

# Telemetry configuration

Only partial configuration will be shown in the examples below. Refer to my [previous article](https://xrdocs.io/telemetry/tutorials/telemetry-stack-update-qos-interface-statistics-example/) and to the [Github repository](https://github.com/RomainCyr/tig-stack-qos-interface-statistics) for full configuration files. 

## Dial-out method

In the dial-out scenario, the TLS server is the Telegraf collector and the client is the router. Therefore, the routers must be able to verify the Telegraf certificate.

### XR configuration

1. The first step is to copy the root CA certifate to the router. In the example below, this is done using SCP.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco/pki# <span style="background-color:yellow;">scp CA.pem cisco@R1.routers.lab:/harddisk:/</span>
    Password: 
    CA.pem                                                                                                                                                                    100% 1220   425.4KB/s   00:00    
    cisco@server1:/home/cisco/pki#
    </code></pre></div>
2. Then, the certificate must be copied to `/misc/config/grpc/dialout/dialout.pem`. Note that the filename is important, it must be `dialout.pem`. This root certificate will be used to verify the Telegraf certificate.
    <div class="highlighter-rouge"><pre class="highlight"><code>RP/0/RP0/CPU0:R1#<span style="background-color:yellow;">run cp /harddisk\:/CA.pem /misc/config/grpc/dialout/dialout.pem</span> 
    Thu Aug 31 08:23:32.316 UTC

    RP/0/RP0/CPU0:R1#
    </code></pre></div>
3. Below is a configuration example for Telemetry dial-out with TLS. The tls-hostname attribute is important and must correspond to the subject name of the Telegraf certificate.
    <div class="highlighter-rouge"><pre class="highlight"><code>telemetry model-driven
     destination-group TIG
      vrf MGMT
      address-family ipv4 192.168.122.1 port 57500
       encoding self-describing-gpb
       <span style="background-color:#F0FFFF;">protocol grpc tls-hostname telegraf.lab</span>
    </code></pre></div>
4. The `emsd` process may need to be restarted to include the `dialout.pem` certificate
    <div class="highlighter-rouge"><pre class="highlight"><code>RP/0/RP0/CPU0:R1#<span style="background-color:yellow;">process restart emsd</span>
    Thu Aug 31 15:45:27.278 UTC
    RP/0/RP0/CPU0:R1#
    </code></pre></div>

## Telegraf Configuration

The input plugin `inputs.cisco_telemetry_mdt` is used for the dial-out method. To enable TLS, the private key and certificate files to use must be configured.

<div class="highlighter-rouge"><pre class="highlight"><code>[[inputs.cisco_telemetry_mdt]]
 ## Telemetry transport can be "tcp" or "grpc".  TLS is only supported when
 ## using the grpc transport.
 transport = "grpc"

 ## Address and port to host telemetry listener
 service_address = ":57500"

 ## Grpc Maximum Message Size, default is 4MB, increase the size. This is
 ## stored as a uint32, and limited to 4294967295.
 max_msg_size = 4000000

 ## Enable TLS; grpc transport only.
 <span style="background-color:#F0FFFF;">tls_cert = "/etc/telegraf/cert.pem"</span>
 <span style="background-color:#F0FFFF;">tls_key = "/etc/telegraf/key.pem"</span>
</code></pre></div>

## Dial-in method

In the dial-in scenario, the routers are acting as TLS servers and the Telegraf collector is the client. Therefore, the Telegraf collector must be able to verify the routers certificates. 

If mutual authentication is used, the client certificate will also be verified. This means that the Telegraf collector will send its certificate that must be verified by the routers.

### XR configuration

TLS is enabled by default when gRPC is configured. One must ensure that `no-tls` is not configured.
<div class="highlighter-rouge"><pre class="highlight"><code>grpc
 vrf MGMT
 port 57400
</code></pre></div>

In addition, if client authentication is required, `tls-mutual` must be added.
<div class="highlighter-rouge"><pre class="highlight"><code>grpc
 vrf MGMT
 port 57400
 <span style="background-color:#F0FFFF;">tls-mutual</span>
</code></pre></div>

When gRPC with TLS is enabled, XR will automatically generate a self-signed certificate with a root CA certificate at the following path `/misc/config/grpc`.

<div class="highlighter-rouge"><pre class="highlight"><code>RP/0/RP0/CPU0:R1#run ls /misc/config/grpc
Thu Aug 31 15:33:08.045 UTC
ca.cert  dialout  ems.key  ems.pem

RP/0/RP0/CPU0:R1#
</code></pre></div>

This self-signed certificate could be used by providing the root certificate (`ca.cert`) to the Telegraf collector for verification. However, in this example, it will be replaced by the routers certificate created above.

If mutual authentication is used the `ca.cert` must also be replaced. Indeed, this is the root certificate used for client certificate verification.

When replacing those files ensure that their names are kept identical.


1. The private key and the certificate are copied onto the routers. The root CA certificate is also copied for mutual authentication. In the example below, this is done using SCP.
    <div class="highlighter-rouge"><pre class="highlight"><code>cisco@server1:/home/cisco/pki# <span style="background-color:yellow;">scp routers.lab.key cisco@R1.routers.lab:/harddisk:/</span>
    Password: 
    routers.lab.key                                                                                                                                                           100% 1679   315.9KB/s   00:00    
    cisco@server1:/home/cisco/pki# <span style="background-color:yellow;">scp routers.lab.pem cisco@R1.routers.lab:/harddisk:/</span>
    Password: 
    routers.lab.pem                                                                                                                                                           100% 1554   609.5KB/s   00:00    
    cisco@server1:/home/cisco/pki# <span style="background-color:yellow;">scp CA.pem cisco@R1.routers.lab:/harddisk:/</span>
    Password: 
    CA.pem                                                                                                                                                                    100% 1220   415.9KB/s   00:00    
    cisco@server1:/home/cisco/pki# 
    </code></pre></div>

2. Then, those files must be copied to the directory `/misc/config/grpc/`.
    <div class="highlighter-rouge"><pre class="highlight"><code>RP/0/RP0/CPU0:R1#<span style="background-color:yellow;">run cp /harddisk:/routers.lab.key /misc/config/grpc/ems.key</span>
    Thu Aug 31 15:43:48.106 UTC

    RP/0/RP0/CPU0:R1#<span style="background-color:yellow;">run cp /harddisk:/routers.lab.pem /misc/config/grpc/ems.pem</span>
    Thu Aug 31 15:44:01.619 UTC

    RP/0/RP0/CPU0:R1#<span style="background-color:yellow;">run cp /harddisk:/CA.pem /misc/config/grpc/ca.cert</span>
    Thu Aug 31 15:44:13.242 UTC

    RP/0/RP0/CPU0:R1#
    </code></pre></div>
3. Once all the files are copied, the `emsd` process must be restart to include those file changes
    <div class="highlighter-rouge"><pre class="highlight"><code>RP/0/RP0/CPU0:R1#<span style="background-color:yellow;">process restart emsd</span>
    Thu Aug 31 15:45:27.278 UTC
    RP/0/RP0/CPU0:R1#
    </code></pre></div>

## Telegraf Configuration

The input plugin `inputs.gnmi` is used for the dial-in method. To enable TLS, the `tls_enabled` attribute must be set to `true` and the root CA certificate, used for verifying the routers certificates, must be provided. Optionally, if mutual authentication is enabled, the Telegraf private key and certificate must be provided.
<div class="highlighter-rouge"><pre class="highlight"><code># gNMI telemetry input plugin
[[inputs.gnmi]]
  ## Address and port of the gNMI GRPC server
  addresses = ["R0.routers.lab:57400","R1.routers.lab:57400"]

  ## define credentials
  username = "cisco"
  password = "cisco123"

  ## gNMI encoding requested (one of: "proto", "json", "json_ietf", "bytes")
  encoding = "json_ietf"

  ## redial in case of failures after
  redial = "10s"

  ## gRPC Maximum Message Size
  # max_msg_size = "4MB"

  ## Enable to get the canonical path as field-name
  # canonical_field_names = false

  ## Remove leading slashes and dots in field-name
  # trim_field_names = false

  ## enable client-side TLS and define CA to authenticate the device
  <span style="background-color:#F0FFFF;">enable_tls = true</span>
  <span style="background-color:#F0FFFF;">tls_ca = "/etc/telegraf/CA.pem"</span>
  ## Minimal TLS version to accept by the client
  # tls_min_version = "TLS12"
  ## Use TLS but skip chain & host verification
  # insecure_skip_verify = true

  ## define client-side TLS certificate & key to authenticate to the device
  <span style="background-color:#F0FFFF;">tls_cert = "/etc/telegraf/cert.pem"</span>
  <span style="background-color:#F0FFFF;">tls_key = "/etc/telegraf/key.pem"</span>
</code></pre></div>

# Verification

The command `show telemetry model-driven destination` displays the current status of TLS.

When TLS is not enabled, the TLS state is shown as `False`.
<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#<span style="background-color: yellow">show telemetry model-driven destination</span>
Thu Aug 31 15:02:24.837 UTC
  Group Id         Sub                          IP                                            Port    Encoding            Transport   State       
  ------------------------------------------------------------------------------------------------------------------------------------------
  TIG              TIG                          192.168.122.1                                 57500   self-describing-gpb grpc        Active      
      <span style="background-color: #7CFC00;">TLS:                  False</span>
  Collection statistics:
    Maximum tokens                   : 4000
    Event tokens                     : 750
    Cadence tokens                   : 740
    Token processed at               : 
    Cadence token advertised at      : 
    Event token advertised at        : 
    GNMI initial synchronization time: 
    Pending queue size               : 0
    Pending queue memory size (bytes): 0
    Processed events                 : 0
    Collection tokens                : 740

RP/0/RP0/CPU0:R1#
</code></pre></div>

For the dial-out method, when TLS is enabled the collector hostname is displayed.

<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#<span style="background-color: yellow">show telemetry model-driven destination</span>
Fri Sep  1 15:28:45.796 UTC
  Group Id         Sub                          IP                                            Port    Encoding            Transport   State       
  ------------------------------------------------------------------------------------------------------------------------------------------
  TIG              TIG                          192.168.122.1                                 57500   self-describing-gpb grpc        Active      
    <span style="background-color: #7CFC00;">TLS :                 telegraf.lab</span>
  Collection statistics:
    Maximum tokens                   : 4000
    Event tokens                     : 750
    Cadence tokens                   : 748
    Token processed at               : 
    Cadence token advertised at      : 
    Event token advertised at        : 
    GNMI initial synchronization time: 
    Pending queue size               : 0
    Pending queue memory size (bytes): 0
    Processed events                 : 0
    Collection tokens                : 748

RP/0/RP0/CPU0:R1#
</code></pre></div>

When the dial-in method is used with TLS, it displays `True`. 
<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#show telemetry model-driven destination 
Fri Sep  1 15:24:09.491 UTC
  Group Id         Sub                          IP                                            Port    Encoding            Transport   State       
  ------------------------------------------------------------------------------------------------------------------------------------------
  GNMI_1003        GNMI__5033885546243418454    192.168.122.1                                 55110   gnmi-json           dialin      Active      
    <span style="background-color: #7CFC00;">TLS :                 True</span>
  Collection statistics:
    Maximum tokens                   : 4000
    Event tokens                     : 750
    Cadence tokens                   : 735
    Token processed at               : 
    Cadence token advertised at      : 
    Event token advertised at        : 
    GNMI initial synchronization time: 2023-09-01 15:23:59.284059 +0000
    Pending queue size               : 0
    Pending queue memory size (bytes): 0
    Processed events                 : 0
    Collection tokens                : 735

RP/0/RP0/CPU0:R1#
</code></pre></div>

If mutual authentication is used, the output varies slightly.
<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#show telemetry model-driven destination 
Fri Sep  1 15:25:01.714 UTC
  Group Id         Sub                          IP                                            Port    Encoding            Transport   State       
  ------------------------------------------------------------------------------------------------------------------------------------------
  GNMI_1005        GNMI__5033885546243418454    192.168.122.1                                 46122   gnmi-json           dialin      Active      
    <span style="background-color: #7CFC00;">TLS-mutual:           True</span>
  Collection statistics:
    Maximum tokens                   : 4000
    Event tokens                     : 750
    Cadence tokens                   : 742
    Token processed at               : 
    Cadence token advertised at      : 
    Event token advertised at        : 
    GNMI initial synchronization time: 2023-09-01 15:25:01.524059 +0000
    Pending queue size               : 0
    Pending queue memory size (bytes): 0
    Processed events                 : 0
    Collection tokens                : 742

RP/0/RP0/CPU0:R1#
</code></pre></div>

# Common issues

TLS issues are often related to configuration mistakes or certificates problems. Most of the errors can be found either in the Telegraf logs or in the gRPC or Telemetry traces.

The logs of the Telegraf container can be read with the following command `docker logs telegraf`. Debugs can be activated in Telegraf with the following config attribute `debug = true`.

On XR, most certificates errors can be found in the following traces:
 - `show grpc trace ems`
 - `show telemetry model-driven trace go-info`
 
More traces can be found with the following commands:
 - `show grpc trace {all|ems|yfed|yfw}`
 - `show telemetry model-driven trace {all|backend-err|backend-sub|backend-timer|config|error|event|go-info|startup|subdb|timer}`

Those traces can be cleared with the following:
 - `clear grpc trace {all|ems|yfed|yfw}`
 - `clear telemetry model-driven trace {all|backend-err|backend-sub|backend-timer|config|error|event|go-info|startup|subdb|timer}`

## Dial-out

### No certificate found

**Symptoms:** The Telemetry connection never established because the root certificate to verify the Telegraf collector certificate is not found. The following error can be found in the Telemetry traces:
<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#<span style="background-color: yellow;">show telemetry model-driven trace go-info</span>
Sep  5 13:58:37.868 m2m/mdt/go-info 0/RP0/CPU0 t25679  1410 [mdt_go_trace_info]: mdtConnEstablish:164 1: Dialing out to MGMT#192.168.122.1:57500, req 501
Sep  5 13:58:37.868 m2m/mdt/go-info 0/RP0/CPU0 t25679  1411 [mdt_go_trace_error]: mdtConnEstablish:213 <span style="background-color: #7CFC00;">Configured TLS, but no certificate exists for 192.168.122.1:57500</span>
RP/0/RP0/CPU0:R1#
</code></pre></div>

**Solution:** Verify that the root certificate was correctly copied to `/misc/config/grpc/dialout/dialout.pem`. The filename is important it MUST be `dialout.pem`. Do not forget to restart the emsd process once the certificate is changed.

### Certificate signed by unknown authority

**Symptoms:** The Telegraf collector certificate cannot be verified and the connection fails at TLS handshake. The following error can be found in the Telemetry traces: 
<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#<span style="background-color: yellow;">show telemetry model-driven trace go-info</span>
Sep  5 14:42:25.428 m2m/mdt/go-info 0/RP0/CPU0 t27179  2679 [mdt_go_trace_info]: mdtConnEstablish:164 1: Dialing out to MGMT#192.168.122.1:57500, req 501
Sep  5 14:42:25.428 m2m/mdt/go-info 0/RP0/CPU0 t27179  2680 [mdt_go_trace_info]: mdtConnEstablish:228 1: req 501, TLS is true, cert /misc/config/grpc/dialout/dialout.pem, host telegraf.lab, compression .
Sep  5 14:42:25.443 m2m/mdt/go-info 0/RP0/CPU0 t27220  2681 [mdt_go_trace_info]: mdtConnEstablish:267 dial: target 192.168.122.1:57500
Sep  5 14:42:25.451 m2m/mdt/go-info 0/RP0/CPU0 t27179  2682 [mdt_go_trace_info]: mdtDialer:271 1: namespace MGMT, args MGMT#192.168.122.1:57500
Sep  5 14:42:25.451 m2m/mdt/go-info 0/RP0/CPU0 t27179  2683 [mdt_go_trace_info]: getAddDialerChanWithLock:164 1: Start a dialer for namespace MGMT
Sep  5 14:42:25.453 m2m/mdt/go-info 0/RP0/CPU0 t27179  2684 [mdt_go_trace_info]: mdtDialer:284 1: Create dialerChan 0xc000010028 for MGMT
Sep  5 14:42:25.737 m2m/mdt/go-info 0/RP0/CPU0 t27220  2685 [mdt_go_trace_error]: mdtConnEstablish:294 1: grpc service call failed, ReqId 501, MGMT#192.168.122.1:57500, rpc error: code = Unavailable desc = connection error: <span style="background-color: #7CFC00;">desc = "transport: authentication handshake failed: x509: certificate signed by unknown authority"</span>
RP/0/RP0/CPU0:R1#
</code></pre></div>

**Solution:** Verify the root certificate copied on the routers as well as the Telegraf certificate on the collector. The root certificate must be able to verify that the Telegraf certificate is valid. This can be verified with the following openssl command: `openssl verify -CAfile CA.pem telegraf.lab.pem `

## Dial-in

### Failed to verify certificate

**Symptoms:** The Telegraf collector cannot verify the router certificate. The following error can be found in the Telegraf logs: 
<div class="highlighter-rouge"><pre class="highlight"><code>
cisco@server1:/home/cisco# <span style="background-color: yellow;">docker logs telegraf</span>
2023-09-05T15:05:54Z I! Loading config: /etc/telegraf/telegraf.conf
2023-09-05T15:05:54Z W! DeprecationWarning: Option "enable_tls" of plugin "inputs.gnmi" deprecated since version 1.27.0 and will be removed in 2.0.0: use 'tls_enable' instead
2023-09-05T15:05:54Z I! Starting Telegraf 1.27.3
2023-09-05T15:05:54Z I! Available plugins: 237 inputs, 9 aggregators, 28 processors, 23 parsers, 59 outputs, 4 secret-stores
2023-09-05T15:05:54Z I! Loaded inputs: gnmi
2023-09-05T15:05:54Z I! Loaded aggregators: 
2023-09-05T15:05:54Z I! Loaded processors: converter regex starlark strings (2x)
2023-09-05T15:05:54Z I! Loaded secretstores: 
2023-09-05T15:05:54Z I! Loaded outputs: influxdb_v2
2023-09-05T15:05:54Z I! Tags enabled: host=85d46ba177ba
2023-09-05T15:05:54Z W! Deprecated inputs: 0 and 1 options
2023-09-05T15:05:54Z I! [agent] Config: Interval:10s, Quiet:false, Hostname:"85d46ba177ba", Flush Interval:10s
2023-09-05T15:05:54Z E! [inputs.gnmi] Error in plugin: failed to setup subscription: rpc error: code = Unavailable desc = connection error: <span style="background-color: #7CFC00;">desc = "transport: authentication handshake failed: tls: failed to verify certificate: x509: certificate is valid for ems.cisco.com, not R1.routers.lab"</span>
</code></pre></div>

**Solution:** Verify that the router certificate and private key were correctly copied to `/misc/config/grpc/`. The filenames are important, it MUST be `ems.pem` and `ems.key`. Do not forget to restart the emsd process once the certificate is changed.

### Client didn't provide a certificate

**Symptoms:** TLS mutual authentication is enabled and the Telegraf collector certificate cannot be verified by the router. The following error can be found in the gRPC traces:
<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:R1#<span style="background-color: yellow;">show grpc trace ems</span>
Tue Sep  5 15:17:18.121 UTC
Sep  5 15:16:59.700 ems/grpc 0/RP0/CPU0 t1449 EMS-GRPC: [core] grpc: Server.Serve failed to complete security handshake from "192.168.122.1:45394": tls: client didn't provide a certificate
Sep  5 15:17:09.706 ems/info 0/RP0/CPU0 t1449 EMS_INFO: Accept:237 Client connected [tcp]
Sep  5 15:17:09.714 ems/grpc 0/RP0/CPU0 t1420 EMS-GRPC: [core] grpc: Server.Serve failed to complete security handshake from "192.168.122.1:48360": <span style="background-color: #7CFC00;">tls: client didn't provide a certificate</span>
RP/0/RP0/CPU0:R1#
</code></pre></div>

The following error can also be found in the Telegraf logs: 
<div class="highlighter-rouge"><pre class="highlight"><code>
cisco@server1:/home/cisco# <span style="background-color: yellow;">docker logs telegraf</span>
2023-09-05T15:05:54Z I! Loading config: /etc/telegraf/telegraf.conf
2023-09-05T15:05:54Z W! DeprecationWarning: Option "enable_tls" of plugin "inputs.gnmi" deprecated since version 1.27.0 and will be removed in 2.0.0: use 'tls_enable' instead
2023-09-05T15:05:54Z I! Starting Telegraf 1.27.3
2023-09-05T15:05:54Z I! Available plugins: 237 inputs, 9 aggregators, 28 processors, 23 parsers, 59 outputs, 4 secret-stores
2023-09-05T15:05:54Z I! Loaded inputs: gnmi
2023-09-05T15:05:54Z I! Loaded aggregators: 
2023-09-05T15:05:54Z I! Loaded processors: converter regex starlark strings (2x)
2023-09-05T15:05:54Z I! Loaded secretstores: 
2023-09-05T15:05:54Z I! Loaded outputs: influxdb_v2
2023-09-05T15:05:54Z I! Tags enabled: host=85d46ba177ba
2023-09-05T15:05:54Z W! Deprecated inputs: 0 and 1 options
2023-09-05T15:05:54Z I! [agent] Config: Interval:10s, Quiet:false, Hostname:"85d46ba177ba", Flush Interval:10s
2023-09-05T15:06:11Z E! [inputs.gnmi] Error in plugin: failed to setup subscription: rpc error: code = Unavailable <span style="background-color: #7CFC00;">desc = connection error: desc = "transport: authentication handshake failed: remote error: tls: bad certificate"</span>
</code></pre></div>

**Solution:** Verify that the Telegraf collector certificate can be verified by the root certificate on the router. The root certificate must be present at the following path `/misc/config/grpc/ca.cert`. The filename is important it MUST be `ca.cert`
