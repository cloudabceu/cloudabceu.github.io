---
layout: post
title: Setup Nginx reverse proxy for Apache CloudStack (Part 2 - VM Console Proxy)
toc: true
categories: [CloudStack]
---

Apache CloudStack provides VM console via a System VM which is named Console Proxy VM (aka CPVM). This article explains how CPVM works, how to secure CPVM and configure nginx for CPVM.

<!--more-->

## How CPVM works

WebSocket is standardized by the IETF as RFC 6455 in 2011. It provide a mechanism for browser-based applications that need two-way communication with servers that does not rely on opening multiple HTTP connections (e.g., using XMLHttpRequest or iframes and long polling).  More information can be found at https://tools.ietf.org/html/rfc6455

The following workflow explains how web users access the VM console via websocket.
![How to Access VM console]({{ "/images/2024-03-20_21-16-How-CPVM-works.png" | absolute_url }}){:width="100%"}{:.glightbox}

- Firstly, the web client sends HTTP request to a web server and get the HTTP response which supports VNC display and websocket connections.
- Secondly, the web client sends request to websocket proxy to open websocket connection.
- Then the websocket proxy starts VNC handshake and VNC authentication if needed.
- The websocket connection between web client and VNC server via the websocket proxy is established.
- The VNC console is displayed on the UI

## CPVM components

The Console Proxy VM (CPVM) has two main components
![Console Proxy VM components]({{ "/images/2024-03-20_20-47-CPVM-components.png" | absolute_url }}){:width="100%"}{:.glightbox}

- NoVNC. noVNC is both a HTML VNC client JavaScript library and an application built on top of that library. noVNC runs well in any modern browser including mobile browsers (iOS and Android). Refer to [NoVNC](https://github.com/novnc/noVNC)
- Jetty Websocket Server. CPVM acts as a websocket proxy between web client and VNC port of VM (on KVM, XenServer/XCP and VMware 6.5-), or websocket connection (on VMware 7.0+).

Please note,
- If SSL is disabled in CPVM, the NoVNC port is `80` and Websocket port is `8080`
- If SSL is enabled in CPVM, the NoVNC port is `443` and Websocket port is `8443`

## Upload SSL certificates for System VMs

Since secure nginx HTTPS server will be configured for VM console, the SSL certificate needs to be uploaded in Apache CloudStack, the System VMs need to be configured as well. Please refer to [Securing CloudStack 4.11 with HTTPS/TLS](https://www.shapeblue.com/securing-cloudstack-4-11-with-https-tls/)

| **Configuration name** | **value** |
| -------- | ------- |
| consoleproxy.url.domain | acsconsole.cloudabc.eu |
| consoleproxy.sslEnabled | true |

Please restart `cloudstack-management` service, and stop/start the CPVM.

If you do not have SSL certificate, you can generate a Let's Encrypt certificate via `certbot`, please refer to the previous blog [Acquire Let’s Encrypt Certificate by certbot](/cloudstack/2024/03/18/nginx-for-cloudstack-part1/#acquire-lets-encrypt-certificate-by-certbot)

Navigate to `Infrastructure` and click the icon `SSL certificates`, a dialog is pop-up.
![Upload SSL certificate]({{ "/images/2024-03-20_20-56-upload-ssl-certificates.png" | absolute_url }}){:width="100%"}{:.glightbox}

Wait until the System VMs are Running and agent state is Up.

## Setup Nginx for Console Proxy VM

The nginx config has two parts, (1) for NoVNC server (port 443); (2) for Websocket Proxy (port 8443).

```
[root@mgmt1 ~]# cat /etc/nginx/conf.d/acsconsole.conf 
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen              8443 ssl http2;
    server_name         acsconsole.cloudabc.eu;
    ssl_certificate     /etc/letsencrypt/live/cloudabc.eu/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cloudabc.eu/privkey.pem;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_verify_client   off;

    location /websockify {
        proxy_pass https://10.0.53.173:8443/websockify;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
    }
}

server {
    listen              443 ssl http2;
    server_name         acsconsole.cloudabc.eu;
    ssl_certificate     /etc/letsencrypt/live/cloudabc.eu/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cloudabc.eu/privkey.pem;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_verify_client   off;

    location / {
        proxy_pass https://10.0.53.173;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_host;
    }
}

```

Refer to [WebSocket proxying](https://nginx.org/en/docs/http/websocket.html) how to setup nginx for websocket.

## Example: VM console

Look at the URL of the VM console, it uses `https://acsconsole.cloudabc.eu` and (websocket) `port` is set to `8443`

![VM Console]({{ "/images/2024-03-20_22-11-VM-console.png" | absolute_url }}){:width="100%"}{:.glightbox}

