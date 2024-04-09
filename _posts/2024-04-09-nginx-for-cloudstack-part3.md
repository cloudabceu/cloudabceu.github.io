---
layout: post
title: Setup Nginx reverse proxy for Apache CloudStack (Part 3 - Secondary Storage VM)
toc: true
categories: [CloudStack]
---

Apache CloudStack provides the register/download/upload features via a System VM which is named Secondary Storage VM (aka SSVM). This article explains how to configure nginx for SSVM.

<!--more-->

## Supported features by SSVM

- Register template/ISO
- Upload template/ISO from local
- Download template/ISO
- Upload volume from local
- Download volume

## Upload SSL certificates for System VMs

Please refer to the previous blog [Upload SSL certificates for System VMs](/cloudstack/2024/03/21/nginx-for-cloudstack-part2/#upload-ssl-certificates-for-system-vms)

| **Configuration name** | **value** |
| -------- | ------- |
| secstorage.ssl.cert.domain | acssec.cloudabc.eu |
| secstorage.encrypt.copy    | true |

Restart the cloudstack-management service and destroy running SSVMs.

## Setup Nginx for SSVM

The nginx config is as below

```
[root@mgmt1 ~]# cat /etc/nginx/conf.d/acssec.conf 
server {
    listen              443 ssl http2;
    server_name         acssec.cloudabc.eu;
    ssl_certificate     /etc/letsencrypt/live/cloudabc.eu/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cloudabc.eu/privkey.pem;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_verify_client   off;

    location / {
        proxy_set_header Host $http_host;

        client_max_body_size 0;
        client_body_buffer_size 8k;

        proxy_cache off;
        proxy_buffering off;
        proxy_max_temp_file_size 0;
        proxy_request_buffering off;
        proxy_redirect off;

        proxy_set_header X-signature $http_x_signature;
        proxy_set_header X-metadata $http_x_metadata;
        proxy_set_header X-expires $http_x_expires;

        if ($request_uri ~ "/upload/(.*)") {
            set $uuid         "$1";
            proxy_pass https://10.0.53.174/upload/$uuid;
            break;
        }

        proxy_pass https://10.0.53.174;
    }
}
```

