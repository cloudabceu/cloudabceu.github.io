---
layout: post
title: Setup Nginx reverse proxy for Apache CloudStack (Part 1 - Management server)
toc: true
categories: [CloudStack]
---

Reverse proxy is widely used to access services in internal sites, or forward HTTPS request to HTTP server in the backend. This series of blogs will introduce how to setup ngix reverse proxy for Apache CloudStack services, including Management server, Console Proxy VM and Secondary Storage VM.

<!--more-->

## Get Let's Encrypt certificates

As the first step, we need to acquire a certificate which will be used in nginx and Cloudstack system VMs. If you already have certificates, you can skip this section.
Please replace `cloudabc.eu` with your own domain name.

### Acquire Let's Encrypt Certificate by certbot

```
yum install -y certbot || apt install -y certbot
certbot -d *.cloudabc.eu --manual --preferred-challenges dns certonly
```

We will see the following message
```
Please deploy a DNS TXT record under the name:

_acme-challenge.cloudabc.eu.

with the following value:

wIBXwsEw2234VawuxOscTqxfv7nOLAhl2r513Tcw2Yg

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.cloudabc.eu.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

Go to the domain provider, and add a text record like below
![Text record for acme-challenge.cloudabc.eu]({{ "/images/2024-03-18_20-57-domain-acme-challenge.png" | absolute_url }}){:width="100%"}{:.glightbox}

Open another session and check if the text record is propogated.
```
$ nslookup -type=TXT _acme-challenge.cloudabc.eu
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
_acme-challenge.cloudabc.eu	text = "wIBXwsEw2234VawuxOscTqxfv7nOLAhl2r513Tcw2Yg"

Authoritative answers can be found from:
```
Once the text record is available (see above), press Enter in the terminal.
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/cloudabc.eu/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/cloudabc.eu/privkey.pem
This certificate expires on 2024-06-15.
These files will be updated when the certificate renews.

NEXT STEPS:
- This certificate will not be renewed automatically. Autorenewal of --manual certificates requires the use of an authentication hook script (--manual-auth-hook) but one was not provided. To renew this certificate, repeat this same certbot command before the certificate's expiry date.
```

### Let's Encrypt Certificates and private key
The LetsEncrypt directory has the following files
```
[root@mgmt1 ~]# ls -l /etc/letsencrypt/live/cloudabc.eu/
total 4
lrwxrwxrwx. 1 root root  35 Mar 17 07:54 cert.pem -> ../../archive/cloudabc.eu/cert1.pem
lrwxrwxrwx. 1 root root  36 Mar 17 07:54 chain.pem -> ../../archive/cloudabc.eu/chain1.pem
lrwxrwxrwx. 1 root root  40 Mar 17 07:54 fullchain.pem -> ../../archive/cloudabc.eu/fullchain1.pem
lrwxrwxrwx. 1 root root  38 Mar 17 07:54 privkey.pem -> ../../archive/cloudabc.eu/privkey1.pem
-rw-r--r--. 1 root root 692 Mar 17 07:54 README
```

- cert.pem: The certificate file
- chain.pem: The CA certificate
- fullchain.pem: The combination of `cert.pem` and `chain.pem`
- privkey.pem: The private key of the certificate

Notes
- The Text record can be removed when certificates are done.
- To automatically renew the certificates, please refer to [Certbot DNS plugins](https://eff-certbot.readthedocs.io/en/latest/using.html#dns-plugins).

## Setup Nginx for CloudStack management server

Normally the CloudStack management server runs internally with HTTP port 8080 and path `/client`. To access the CloudStack management server, add a nginx configuration like below
```
[root@mgmt1 ~]# cat /etc/nginx/conf.d/acsmgmt.conf 
server {
    listen              443 ssl http2;
    server_name         acsmgmt.cloudabc.eu;
    ssl_certificate     /etc/letsencrypt/live/cloudabc.eu/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cloudabc.eu/privkey.pem;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_verify_client   off;

    rewrite ^/index.(.*)$ /client permanent;

    location /client {
        proxy_pass http://<MGMT IP>:8080/client;
        proxy_set_header Host $http_host;
        proxy_set_header X_FORWARDED_PROTO https;
    }
}
```

When the domain name is available, access `https://acsmgmt.cloudabc.eu/` or `https://acsmgmt.cloudabc.eu/client`, the same CloudStack login page is displayed.

![CloudStack login]({{ "/images/2024-03-18_21-13-nginx-CloudStack-login.png" | absolute_url }}){:width="100%"}{:.glightbox}

## Part 2: Setup Nginx for Console Proxy VM (To be added)

## Part 3: Setup Nginx for Secondary Storage VM (To be added)
