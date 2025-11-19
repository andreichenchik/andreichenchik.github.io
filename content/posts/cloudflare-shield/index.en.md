---
title: Cloudflare proxy for my Dokku server
date: 2025-08-17T10:00:00+02:00
draft: false
description: 'How I put free Cloudflare proxy in front of my tiny Dokku box, got HTTPS, and locked the server down with a firewall.'
summary: 'Hide a Dokku app behind Cloudflare: move the domain, turn on the proxy, issue an origin certificate, and block direct traffic with a firewall.'
tags: ["infra", "cloudflare", "dokku", "hetzner"]
ShowToc: true
TocOpen: false
cover:
  image: cover.png
---

This is a follow-up to the [minimal Dokku server](/posts/minimal-dokku-setup/) story. I pulled this part out so the original guide stays focused, and we can keep iterating on the same Hello World backend.

The `helloworld` app already [runs in production](https://helloworld.chenchik.me); the server is alive and well. Now I want extra headroom for performance and security, so I’m placing **Cloudflare** between users and the VPS. It’s free, adds caching and HTTP/2, and filters most junk traffic before it ever touches Dokku.

## Prerequisites

- a registered domain you can transfer into [Cloudflare](https://cloudflare.com);
- a Cloudflare account;
- a server with Dokku and the `helloworld` app;
- access to Hetzner Firewall (or your provider’s firewall) so only Cloudflare IPs can reach the box.

## Step-by-step

### 1. Move the domain to Cloudflare

First, give Cloudflare control over DNS. It’s one of the fastest and most reliable DNS platforms, and the base plan is free, so the transfer itself is already an upgrade. In the dashboard I create a new zone and follow the wizard. Every registrar has its own flow, so Cloudflare keeps the official instructions here: https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/ Once the NS records are updated, wait until Cloudflare verifies the zone.

### 2. Turn on the proxy

The zone already contains A-records pointing to my Hetzner IP. By default Cloudflare only answers DNS queries (grey cloud). I need the full proxy, so I switch the `helloworld` record to the orange cloud (Proxied). Now all HTTP/HTTPS traffic flows through Cloudflare and the free edge certificates kick in automatically. At this point the app is already available at `https://helloworld.chenchik.me`.

### 3. Issue an Origin Certificate

This setup works, but managing Let’s Encrypt certificates becomes painful, especially with several apps—Brian explains the issue well: https://bitkidd.dev/posts/use-cloudflare-ssl-certificates-with-dokku To avoid rate limits and keep public certificates away from the private network, I issue an **Origin Certificate** instead. It secures the connection between Cloudflare and the server for up to 15 years. In the dashboard I go to **SSL/TLS → Origin Server**, click “Create Certificate,” and save the private key and certificate as `pet.key` and `pet.crt`.

### 4. Install the certificate in Dokku

Move the files to the server and plug them into Dokku.

1. SSH into the box: `ssh root@<ip>`.
2. Create files with the certificate and key (`nano pet.crt`, `nano pet.key`) and paste the Cloudflare data.
3. Package them, disable the old certificate, and add the new one:

   ```shell
   tar -cvf pet.tar pet.crt pet.key
   dokku letsencrypt:disable helloworld
   dokku certs:add helloworld < pet.tar
   dokku ps:rebuild helloworld
   ```

`certs:add` replaces the certificate, and `ps:rebuild` restarts the app with the updated TLS settings. Cloudflare terminates TLS at the edge and keeps TLS all the way to the server, so the whole chain is encrypted.

Once everything works, I remove the Let’s Encrypt plugin to keep Dokku clean:

```shell
sudo dokku plugin:uninstall letsencrypt
```

### 5. Lock the server with a firewall

Anyone can still bypass Cloudflare and hit the server directly. To prevent that, I enable **Firewall** in Hetzner Cloud and allow inbound 80/443 traffic only from Cloudflare IP ranges: https://www.cloudflare.com/ips/ SSH stays limited to my own addresses. Now the server “sees” the internet exclusively through Cloudflare. Catalin walks through the same setup for Hetzner + Cloudflare: https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/

## Final checks

Confirm that the certificate is in use:

```shell
curl -I https://helloworld.chenchik.me
```

`server: cloudflare` and `cf-cache-status: DYNAMIC` tell me the proxy is in front. Direct requests to the IP already fail on ports 80/443 because the firewall blocks them.

The whole rollout takes about 30–60 minutes. In return you get more performance from caching, free HTTPS, and a shield from random spikes. Next up: adding more features to these little pet projects.

## Sources

- https://bitkidd.dev/posts/use-cloudflare-ssl-certificates-with-dokku
- https://spiffy.tech/dokku-with-lets-encrypt-behind-cloudflare
- https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/
- https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/
