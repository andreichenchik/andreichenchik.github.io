---
title: Minimal Dokku Setup for Side Projects
date: 2025-07-15T10:00:00+02:00
draft: false
description: 'How I assemble a cheap backend for side projects using a VPS, Dokku, and Docker containers.'
summary: 'My minimal stack for mobile side projects: a low-cost Hetzner VPS, Docker containers, and Dokku without infrastructure headaches.'
tags: ["infra", "pet-projects", "dokku", "hetzner"]
ShowToc: true
TocOpen: false
cover:
  image: cover.jpg
---

## Why This Is Hard

Every now and then I get back to tinkering with side projects. As a mobile developer I just need the tiniest backend: return some JSON, write a couple of rows to a database, run a simple bit of logic. Nothing heroic. But the moment I open the first Google result and try to follow the “standard” approach, the supposed basic backend turns into a quest.

### Usual Options
I genuinely like **Cloudflare Workers**, but their SDK immediately dictates an edge architecture. Minimal containers on **DigitalOcean** quickly turn into a noticeable bill, and **Fly.io** (plus similar platforms) comes with its own constraints. There is also **Coolify**: it converts your server into a mini-cloud and covers plenty of corner cases, but you pay for that with the complexity of the solution itself. You need a beefy server, so the infrastructure cost climbs again and suddenly looks like something for a small business, not a Sunday experiment.

### I Want Minimalism
I want something so cheap I can forget about it for a year; without quirky limits so any quick-start guide from the internet works as-is; and flexible enough to wrap a backend without thinking about the stack—Python today, JavaScript tomorrow, Swift the day after. Adding another half-dead project shouldn’t bump the bill or break the setup: services should just run side by side without drama.

### The Sweet Spot: Hetzner + Dokku + Dockerfile
- **Hetzner/VPS:** low-cost, reliable, and always on.
- **Containers:** the `Dockerfile` abstracts the stack, so Python, JavaScript, and Swift deploy the same way.
- **Dokku:** a lightweight “manager” for containers on the box—configure once, spin up or update tiny apps with a couple of simple commands.
- **Budget:** roughly €5/month plus a domain (~€1.5/year) to get human-friendly URLs and enable HTTPS.

## Step-by-Step
Here’s how I set up and use the *Hetzner + Dokku + Dockerfile* combo. It takes about an hour and leaves you with your own mini-cloud.

### Renting the VPS
The provider doesn’t really matter—we just need a small Linux box with SSH access. I’ll show it with Hetzner because the price/performance ratio is excellent.

By the end of this step you should have:
- a running Linux server
- a public IP address
- a password or SSH key so you can log in with `ssh root@<ip>`

**Hetzner Cloud server**
1. Open the [cloud console](https://console.hetzner.com/) and create a project if you don’t have one already.
1. Inside the project hit **Create server** and pick a plan. I use `CX23` on x86. There’s also `CAX11` on ARM—[much faster for the same money](https://sparecores.com/compare/hcloud-2vcpus)—but not all software loves ARM, so for a “no-stress” setup I stay on x86.
1. Location: I choose Germany, Falkenstein. Pick whatever region is closest to you ([latency checker](https://hetzner-latency.sliplane.io)).
1. OS: Ubuntu LTS has the most straightforward guides. If you crave minimalism and don’t mind googling a bit, grab Debian for slightly better efficiency.
1. Networking: I add an IPv4 address (+€0.61/month for me) because my ISP still doesn’t do IPv6. You might be able to skip it.
1. SSH: upload your public key (or create a new one) to log in without a password—it makes Dokku setup easier.
1. Skip the remaining toggles (backup, extra disks). You can revisit them later.
1. Hit **Create & Buy now**, wait for the server to boot, and confirm access via `ssh root@<ip> -i ~/.ssh/hetzner` (if SSH complains about permissions, run `chmod 600 ~/.ssh/hetzner`).

Expect to pay around €3.5–4.5/month for this configuration.

### Domain Setup
I buy the cheapest domain I can find—zones like `.xyz` usually cost €1–2/year—and configure DNS right at the registrar. If they don’t provide DNS, I hook up free [Cloudflare DNS](https://www.cloudflare.com/application-services/products/dns/).

1. Open the domain management panel.
1. Create a wildcard record `*.chenchik.me` (A/AAAA) pointing to the server’s IP.

Now every subdomain such as `helloworld.chenchik.me` resolves to my VPS.

### Installing Dokku
The rest of the commands happen on the server: `ssh root@<ip> -i ~/.ssh/hetzner`.

1. (Optional) update the system:
   ```shell
   # Ubuntu update all
   sudo -s -- <<'EOF'
   apt-get update
   apt-get upgrade -y
   apt-get full-upgrade -y
   apt-get autoremove -y
   apt-get autoclean -y
   EOF
   ```
   I usually reboot afterward (`sudo reboot`) and reconnect.
1. Install [Dokku](https://dokku.com). Latest version at the time of writing is 0.36.10, so I pull the official script and pin the tag:
   ```shell
   wget -NP . https://dokku.com/install/v0.36.10/bootstrap.sh
   sudo DOKKU_TAG=v0.36.10 bash bootstrap.sh
   ```
1. Add an SSH key for deployments by reusing the key I already use for the server:
   ```shell
   cat ~/.ssh/authorized_keys | dokku ssh-keys:add admin
   ```
   *Better yet, generate a dedicated key pair for deployments. If you have a public key elsewhere, just pass it directly:*
   ```shell
   echo '<your-public-key>' | dokku ssh-keys:add admin
   ```
1. Set the default domain:
   ```shell
   dokku domains:set-global <your-domain>
   ```
1. Install the Let’s Encrypt plugin, wire up your email, and enable automatic renewals:
   ```shell
   sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
   dokku letsencrypt:set --global email <your-email>
   sudo dokku letsencrypt:cron-job --add
   ```

That’s it—Dokku is ready and we can move on to the first backend.

### First Backend
Time to deploy something. The first two commands run on the server; everything else happens locally.

1. Create an app on the server:
   ```shell
   dokku apps:create helloworld
   ```
1. Enable HTTPS:
   ```shell
   dokku letsencrypt:enable helloworld
   ```
1. Locally, create an empty Git repo and point it at Dokku:
   ```shell
   git init helloworld
   cd helloworld
   git remote add dokku dokku@<ip>:helloworld
   ```
1. Verify the connection (`ssh -T dokku@<ip>` should list commands). If it fails, add a snippet to `~/.ssh/config` so SSH knows which key to use:
   ```
   Host <ip>
     IdentityFile ~/.ssh/hetzner
     AddKeysToAgent yes
     UseKeychain yes
   ```
   After creating the file, run `chmod 600 ~/.ssh/config` to fix permissions.
1. Create a minimal backend:
   ```shell
   cat <<'EOF' > server.js
   const http = require('http');
   http.createServer((_, res) => {
      res.writeHead(200, {'Content-Type':'application/json'});
      res.end('{"message":"hello world"}');
   }).listen(80);
   EOF
   ```
1. Add a tiny `Dockerfile`:
   ```shell
   cat <<'EOF' > Dockerfile
   FROM node:24-alpine
   COPY server.js .
   EXPOSE 80
   CMD ["node", "server.js"]
   EOF
   ```
1. Commit and deploy. As soon as `git push` reaches the server, Dokku builds the image and restarts the app:
   ```shell
   git add -A && git commit -m "first backend"
   git push --set-upstream dokku main
   ```

Done! The app now lives at https://helloworld.chenchik.me

## Sources
- https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/
- https://dokku.com/docs/getting-started/installation/
