---
title: "Adding a persistence layer for side projects"
date: 2025-08-29T10:00:00+02:00
draft: false
description: 'How I run one shared PostgreSQL on Hetzner, hook up all Dokku apps, and send daily dumps to Cloudflare R2.'
summary: 'One PostgreSQL on Dokku plus Cloudflare R2 backups — my baseline persistence for mobile apps.'
tags: ["infra", "postgres", "cloudflare", "dokku"]
ShowToc: true
TocOpen: false
cover:
  image: cover.jpg
---

This continues the [minimal Dokku server](/posts/minimal-dokku-setup/) story — I’m building myself a tiny app platform.

## Why bother with a database

Backends for mobile projects exist to process data off-device, so even the “minimal” backend must store data reliably; otherwise keeping a server online makes no sense. Before another side project, I set up the persistent layer first, then deploy new apps.

## What I picked

Options were SQLite (each service has its own file), several PostgreSQL instances, or one shared instance. I like SQLite, but file management is annoying and I want minimal configuration. So I went with the simplest manageable setup:

- **One PostgreSQL on the server**: least maintenance; if it dies, all services die, but on a single VPS that’s acceptable.
- **One database for all projects**: saves memory and CPU slots; separation lives in tables.
- **Backups**: via the Dokku plugin to external storage out of the box.

## Create the database

Install the PostgreSQL Dokku plugin once:

```shell
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
```

Then create a single shared database (service name `db`) and use separate tables/schemas per project:

```shell
dokku postgres:create db
```

## Bun app example

On the server:

```shell
dokku apps:create fruits
dokku certs:add testapp < chenchik.tar
dokku postgres:link db fruits
```

Reminder: the certificate line is from the [previous post](/posts/cloudflare-shield/); it enables HTTPS. Alternatively, you can use `letsencrypt` like in the original guide.

When you link the DB to an app, Dokku adds `DATABASE_URL`, which lets the app connect. Locally I assemble a minimal Bun API, mainly because Bun ships with `postgres` support. The example is intentionally compact.

Create a folder with two files.

`server.ts`:

```ts
import { serve } from "bun";
import postgres from "postgres";

const sql = postgres(process.env.DATABASE_URL!);

let seeded = false;

async function ensureSeededOnce() {
  if (seeded) return;

  await sql`CREATE TABLE IF NOT EXISTS items (name TEXT PRIMARY KEY)`;

  await sql`
    INSERT INTO items (name)
    VALUES ('apple'), ('another-apple'), ('banana'), ('carrot')
    ON CONFLICT DO NOTHING
  `;

  seeded = true;
}

serve({
  port: 80,
  async fetch(req) {
    const url = new URL(req.url);
    const filter = url.searchParams.get("filter") ?? "";

    await ensureSeededOnce();

    const rows = await sql<{ name: string }[]>`
      SELECT name FROM items
      WHERE name ILIKE ${"%" + filter + "%"}
    `;

    return new Response(JSON.stringify(rows), {
      status: 200,
      headers: { "Content-Type": "application/json" },
    });
  },
});
```

`Dockerfile`:

```dockerfile
FROM oven/bun:alpine
WORKDIR /app
COPY server.ts .
EXPOSE 80
CMD ["bun", "server.ts"]
```

Deploy to Dokku from the folder with these two files:

```shell
git init .
git remote add dokku dokku@<ip>:fruits
git add server.ts Dockerfile
git commit -m "fruits api"
git push --set-upstream dokku main
```

Check: https://fruits.chenchik.me/?filter=app — it works!

## Set up backups to Cloudflare R2

Data is too easy to lose, so I set up backups once and forget about them. The plugin ships with `postgres:backup`, which can send dumps to an AWS S3-compatible bucket. Many providers offer this; the ones I use are Hetzner and Cloudflare. Hetzner has a minimum monthly cost; Cloudflare R2 instead gives 10 GB free, which is plenty for these DBs. If I ever need more, I’ll probably reconfigure the server anyway. To configure backups to R2 I need a few parameters: key, secret, and endpoint (region and signature version are for AWS proper, not relevant here).

Authorize R2:

```shell
dokku postgres:backup-auth db \
  <R2_ACCESS_KEY> \
  <R2_SECRET_KEY> \
  us-east-1 \
  s3v4 \
  <R2_ENDPOINT>
```

You can grab these in the [Cloudflare dashboard](https://dash.cloudflare.com/), Build > Storage & Databases > R2 object storage. On the right, open API Tokens -> Manage. `R2_ENDPOINT` is a full URL—copy it entirely.

Then I create a bucket in R2 (say, `db-backups`) and add a schedule on the server for [0 3 * * *](https://crontab.guru#0_3_*_*_*) — daily at 3 a.m.:

```shell
dokku postgres:backup-schedule db "0 3 * * *" db-backups
```

Manual one-off check:

```shell
dokku postgres:backup db db-backups
```

This stores a logical `pg_dump` in R2; to restore, download the dump and load it back into `db`.

## Takeaways

The simplest possible persistence layer with an automatic backup. 
