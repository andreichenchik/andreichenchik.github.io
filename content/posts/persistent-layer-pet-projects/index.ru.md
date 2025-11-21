---
title: "Добавляю persistent-слой для пет-проектов"
date: 2025-08-29T10:00:00+02:00
draft: false
description: 'Как я поставил один общий PostgreSQL на Hetzner, привязал к нему докку-приложения и настроил ежедневные дампы в Cloudflare R2.'
summary: 'Один PostgreSQL на Dokku плюс бэкап в Cloudflare R2 — мой базовый persistent-слой для мобильных приложений.'
tags: ["infra", "postgres", "cloudflare", "dokku"]
ShowToc: true
TocOpen: false
cover:
  image: cover.jpg
---

Продолжаю историю, начатую с [минимальным Dokku-сервером](/ru/posts/minimal-dokku-setup/), создаю себе платформу для мини приложений. 

## Зачем вообще база

Обычно для мобильных проектов backend нужен для обработки данных вне клиента, поэтому даже «минимальный» бэкенд должен уметь сохранять данные, иначе весь смысл постоянного сервера теряется. Перед очередным пет-проектом надо настроить persistent-слой, а уже потом деплоить новые приложения.

## Что выбрал

Из вариантов были SQLite (в каждом сервисе свой файл), несколько PostgreSQL-инстансов или один общий. Я сам фанат SQLite, но с ним будут сложности менеджмента файлов, а я стремлюсь к минимум конфигурации. Поэтому остановился на простом и управляемом варианте:

- **Один PostgreSQL на сервере**: меньше всего обслуживания; если он падает, все сервисы падают, но на одном VPS это ок.
- **Одна база для всех проектов**: экономлю память и CPU-ресурсы; разграничиваю всё таблицами.
- **Бэкапы**: через докку-плагин на внешний сервер из коробки.

## Создаем базу данных

Докку-плагин для PostgreSQL ставится один раз:

```shell
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
```

Дальше завожу одну общую базу (имя сервиса — `db`) и использую в ней отдельные таблицы/схемы под проекты:

```shell
dokku postgres:create db
```

## Пример приложения на Bun

На сервере:

```shell
dokku apps:create fruits
dokku certs:add testapp < chenchik.tar
dokku postgres:link db fruits
```
Напоминаю: строчка с сертификатами из [предыдущей статьи](/ru/posts/cloudflare-shield/), она добавляет поддержку HTTPS, как альтернатива можно использовать `letsencrypt` как в оригинальной статье.

При связывании базы с приложением dokku добавит переменную окружения `DATABASE_URL`, которая позволяет подключиться к серверу. Дальше локально собираю минимальный пример API на Bun, просто потому что в нем есть поддержка `postgres`. Пример нарочито компактный.

Создаем папку с двумя файлами.

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

Деплой в докку из папки с двумя файлами выше:

```shell
git init .
git remote add dokku dokku@<ip>:fruits
git add server.ts Dockerfile
git commit -m "fruits api"
git push --set-upstream dokku main
```

Проверяем: https://fruits.chenchik.me/?filter=app, работает!

## Настраиваем бэкап в Cloudflare R2

Если есть данные, их легко потерять, поэтому сразу настраиваю бэкап, чтобы сделать и забыть. У плагина есть стандартная команда `postgres:backup`, с помощью которой можно делать резервные копии в AWS S3-совместимый контейнер. У многих провайдеров есть подобные решения; у тех, в которых я зарегистрирован, — Hetzner и Cloudflare. У Hetzner, к сожалению, есть минимальная ежемесячная стоимость, а у Cloudflare R2 наоборот есть бесплатный лимит в 10 GB, которого для базы хватит с лихвой; если понадобится больше, скорее всего придется иначе конфигурировать и сервер. Для настройки бэкапа в R2 мне нужны несколько параметров: ключ, секрет и endpoint (регион и версия — только для оригинального AWS-хранилища, нас не интересует).

Для авторизации на R2 нужно запустить команду:

```shell
dokku postgres:backup-auth db \
  <R2_ACCESS_KEY> \
  <R2_SECRET_KEY> \
  us-east-1 \
  s3v4 \
  <R2_ENDPOINT>
```

Получить эти параметры можно в [консоли Cloudflare](https://dash.cloudflare.com/), секция Build > Storage & Databases > R2 object storage. В правой части будет API Tokens -> Manage. `R2_ENDPOINT` — полноценный URL, копируйте его полностью.

После этого создаю бакет в R2 (например, `db-backups`) и на сервере настраиваю расписание [0 3 * * *](https://crontab.guru#0_3_*_*_*) ежедневно в 3 утра:

```shell
dokku postgres:backup-schedule db "0 3 * * *" db-backups
```

Можно вручную проверить разово:

```shell
dokku postgres:backup db db-backups
```

Так сохраняется логический `pg_dump` в R2; для восстановления достаточно скачать дамп и восстановить его на сервере.

## Итоги

Максимально простой слой хранения данных с автоматической резервной копией. 
