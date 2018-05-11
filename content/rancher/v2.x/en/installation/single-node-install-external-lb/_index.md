---
title: Single Node Installation with External Load Balancer
weight: 260
---
# Single Node Installation with External Load Balancer

For development environments, we recommend installing Rancher by running a single Docker container. In this installation scenario, you'll deploy Rancher to a Linux host using a single Docker container. Then you will configure an external load balancer to work with Rancher.

>**Note:**
> If you want don't want to use an external load balancer, please see [Single Node Installation]({{< baseurl >}}/rancher/v2.x/en/installation/single-node-install/).

## Overview

Installation of Rancher on a single node with an external load balancer involves multiple procedures. Review this overview to learn about each procedure you need to complete.

1. [Provision Linux Host](#provision-linux-host)
2. [Choose an SSL Option and Install Rancher](#choose-an-ssl-option-and-install-rancher)
3. [Configure Your Load Balancer](#configure-your-load-balancer)

## Provision Linux Host

Provision a single Linux host to launch your {{< product >}} Server.

### Requirements

{{< requirements_os >}}

{{< requirements_hardware >}}

{{< requirements_software >}}

>**Note:**
>- `rancher/rancher` is hosted on [DockerHub](https://hub.docker.com/r/rancher/rancher/tags/). If you don't have access to DockerHub, or you are installing Rancher without an internet connection, refer to [Installing From a Private Registry]({{< baseurl >}}/rancher/v2.x/en/installation/air-gap-installation/install-from-private-registry/).<br/>
>- For a list of other Rancher Server tags available, refer to [Rancher Server Tags]({{< baseurl >}}/rancher/v2.x/en/installation/server-tags/).

## Choose an SSL Option and Install Rancher

For security purposes, SSL (Secure Sockets Layer) is required when using Rancher. SSL secures all Rancher network communication, like when you login or interact with a cluster.

You can choose from the following scenarios:

- [Option 1—Bring Your Own Certificate: Self-Signed](#option-1-bring-your-own-certificate-self-signed)
- [Option 2—Bring Your Own Certificate: Recognized CA](#certificate-signed-by-a-recognized-certificate-authority)

### Option 1—Self-Signed Certificate

If you elect to use a self-signed certificate to encrypt communication, you must install the certificate on your load balancer (which you'll do later) and your Rancher container. Run the docker command to deploy Rancher, pointing it toward your certificate.

**Before you Start:**

Create a Self-Signed Certificate.

- The certificate files must be in [PEM format](#ssl-faq-troubleshooting).

- The certificate files must be in base64.

| Type                         |        Location in container |
| ---------------------------- | ---------------------------: |
| CA certificates file         | /etc/rancher/ssl/cacerts.pem |


**To Install Rancher Using a Self-Signed Cert:**

While running the Docker command to deploy Rancher, point Docker toward your CA certificate file.

```
docker run -d -p 80:80 -p 443:443 \
  -v /etc/your_certificate_directory/cacerts.pem:/etc/rancher/ssl/cacerts.pem \
  rancher/rancher
```

### Certificate signed by a recognized Certificate Authority

If the certificate you want to use on the load balancer/proxy is signed by a recognized Certificate Authority, you will have to remove the default generated CA certificate information in Rancher. This can be done under `Settings` -> `cacerts`, choose `Edit` and remove the contents and click `Save`.

### Instructions for the load balancer or proxy

When using a load balancer or proxy in front of the `rancher/rancher` container, there is no need for the `rancher/rancher` container to redirect port **TCP/80** (HTTP) to port **TCP/443** (HTTPS). By passing the header `X-Forwarded-Proto:
 https` header, this redirect will be disabled.

The load balancer or proxy has to be configured to support the following:

* Support **WebSocket** connections
* **SPDY** / **HTTP/2** support
* Passing/setting the following headers

| Header               | Value                                     | Description                                                  |
| -------------------- | ----------------------------------------- | :----------------------------------------------------------- |
| `Host`               | Hostname that is used to reach Rancher | To identify the server requested by the client               |
| `X-Forwarded-Proto`  | `https`                                   | To identify the protocol that a client used to connect to the load balancer or proxy<br />*If this Header is present, `rancher/rancher` will not redirect HTTP to HTTPS* |
| `X-Forwarded-Port`   | Port used to reach Rancher                | To identify the protocol that client used to connect to the load balancer or proxy |
| `X-Forwarded-For`    | IP of the client connection               | To identify the originating IP address of a client           |


#### Example NGINX configuration

This NGINX configuration is tested on NGINX 1.13 (mainline) and 1.14 (stable)

```
upstream rancher {
    server rancher-server:80;
}

map $http_upgrade $connection_upgrade {
    default Upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name rancher.yourdomain.com;
    ssl_certificate /etc/your_certificate_directory/fullchain.pem;
    ssl_certificate_key /etc/your_certificate_directory/privkey.pem;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://rancher;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # This allows the ability for the execute shell window to remain open for up to 15 minutes. Without this parameter, the default is 1 minute and will automatically close.
        proxy_read_timeout 900s;
    }
}

server {
    listen 80;
    server_name rancher.yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

#### Example HAProxy configuration

#### Example Amazon ELB configuration

## SSL FAQ / Troubleshooting

{{< ssl_faq >}}

## Persisent data

{{< persistentdata >}}