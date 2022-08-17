---
layout: post
title: Adding Let's Encrypt SSL Certificate with Namecheap
summary: A how to on getting a Let's Encrypt SSL certifcate with namecheap
author: nlgotz
largeimage: assets/le+nc.png
---

![Lets Encrypt + Namecheap](/assets/le+nc.png)

With Google Chrome now giving alerts on internally hosted domains not using HTTPS, I figured it was time to start using Let's Encrypt to add SSL to my internally hosted websites.

None of these sites have external access but I don't like seeing the "Not Secure" icon by the URL as that doesn't look great when presenting.

This all assumes that you have API access setup in Namecheap's portal.

## Script to get an SSL certificate

This script has a couple variables that need to be filled in by the end user:

- E-mail
- Namecheap username
- Namecheap API key
- Namecheap Source IP
- Server name (FQDN)

```bash
curl https://get.acme.sh | sh
acme.sh --register-account -m <email>
export NAMECHEAP_USERNAME=<username>
export NAMECHEAP_API_KEY=<api_key>
export NAMECHEAP_SOURCEIP=<source_ip>
acme.sh --issue -d <server-name> --staging --dns dns_namecheap
```

Once you confirm that it works in staging, then you can push it to production:

```bash
acme.sh --issue -d <server-name> --dns dns_namecheap --force
sudo systemctl restart nginx

```

## NGINX Configuration

Below is a partial config that shows how to redirect SSL and how to use the SSL certificates.

```bash
server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;

    server_name _;

    ssl_certificate /root/.acme.sh/<server-name>/fullchain.cer;
    ssl_certificate_key /root/.acme.sh/<server-name>/<server-name>.key;

    client_max_body_size 25m;

}

server {
    # Redirect HTTP traffic to HTTPS
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
}
```

## Wrap up

Hopefully this helps if you're attempting to use Let's Encrypt on internally hosted domains that are managed through Namecheap.

This also works with Proxmox although most of the scripting isn't needed as it can be handled through the Proxmox GUI.
