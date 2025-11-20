---
title: "Добавляю persistent-слой для пет-проектов"
date: 2025-07-20T10:00:00+02:00
draft: false
description: 'Почему решил держать один PostgreSQL на Hetzner и как подключаю его к докку-приложениям с бэкапом в Cloudflare R2.'
summary: 'Один PostgreSQL на докку-сервере и бэкап в Cloudflare R2 — мой базовый persistent-слой для мобилок.'
tags: ["infra", "pet-projects", "postgres"]
ShowToc: true
TocOpen: false
---

## Зачем вообще база

Большинство мобильных фич упираются в хранение: нужно держать список пользователями созданных сущностей, синхронизировать состояние между устройствами, валидировать платежи. Даже «минимальный» бэкенд должен уметь надежно сохранить данные, иначе весь смысл постоянного сервера теряется. Поэтому перед очередным пет-проектом я сначала решил навести порядок с persistent-слоем, а уже потом деплоить новые контейнеры.

## Что выбрал

Из вариантов были SQLite (в каждом сервисе свой файл), несколько PostgreSQL-инстансов или один общий. Остановился на простом и управляемом варианте:

- **Один PostgreSQL на сервере**: меньше всего обслуживания; если он падает, все сервисы падают, но на одном VPS это ок.
- **Одна база для всех проектов**: экономлю память и слоты соединений; разграничиваю всё таблицами (или схемами по желанию).
- **Бэкапы**: логический `pg_dump` через докку-плагин в объектное хранилище.

## Создаем базу данных

Докку-плагин для PostgreSQL ставится один раз:

```shell
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
```

Дальше завожу одну общую базу (имя сервиса — `petdb`) и использую в ней отдельные таблицы/схемы под проекты:

```shell
dokku postgres:create petdb
dokku ps:restart petdb
dokku postgres:info petdb
```

`info` нужен, чтобы убедиться, что сервис жив и слушает стандартный порт.

## Пример приложения на Bun (создаем и деплоим)

На сервере:

```shell
dokku apps:create fruits
dokku postgres:link petdb fruits
```

Линк ставлю сразу при создании, чтобы `DATABASE_URL` уже был в конфиге к моменту первого деплоя. Дальше локально собираю минимальный API на Bun.

`server.ts`:

```ts
import { serve } from "bun";
import postgres from "postgres";

const sql = postgres(process.env.DATABASE_URL!);
let seeded = false;

async function ensureSeededOnce() {
  if (seeded) return;

  await sql`CREATE TABLE IF NOT EXISTS items (name TEXT NOT NULL)`;

  const [{ count }] = await sql<{ count: string }[]>`
    SELECT COUNT(*)::text as count FROM items
  `;

  if (Number(count) === 0) {
    await sql`
      INSERT INTO items (name)
      VALUES ('apple'), ('banana'), ('carrot')
    `;
  }

  seeded = true;
}

serve({
  port: 80,
  async fetch(req) {
    const url = new URL(req.url);
    const filter = url.searchParams.get("filter") ?? "";

    await ensureSeededOnce();

    const rows = await sql<{ name: string }[]>`
      SELECT name FROM items WHERE name ILIKE ${"%" + filter + "%"}
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

Деплой в докку:

```shell
git init fruits && cd fruits
git remote add dokku dokku@<ip>:fruits
git add server.ts Dockerfile
git commit -m "fruits api"
git push --set-upstream dokku main
dokku ps:restart fruits
```

После пуша приложение читает `DATABASE_URL` от `postgres:link` и пишет в базу `petdb`.

## Настраиваем бэкап в Cloudflare R2

R2 совместим с S3, поэтому подходит стандартная команда `postgres:backup-auth`. Мне нужно пять параметров: ключ, секрет, регион, сигнатура, endpoint. Для R2 они выглядят так:

```shell
dokku postgres:backup-auth petdb \
  <R2_ACCESS_KEY> \
  <R2_SECRET_KEY> \
  auto \
  v4 \
  https://<ACCOUNT_ID>.r2.cloudflarestorage.com
```

Дальше создаю бакет в R2 (например, `petdb-backups`) и настраиваю расписание:

```shell
dokku postgres:backup-schedule petdb "0 3 * * *" petdb-backups/$(date +%F)
```

Можно вручную проверить разово:

```shell
dokku postgres:backup petdb petdb-backups/manual-test-$(date +%s)
```

Так сохраняется логический `pg_dump` в R2; для восстановления достаточно скачать дамп и ресторнуть его в `petdb`.

## Итоги

- Один экземпляр PostgreSQL → минимум администрирования; если что-то ломается, чиню один сервис.
- Одна база для всех → экономия ресурсов; изоляцию держу таблицами или схемами.
- Автоматический `pg_dump` в R2 → ключевой слой защиты от дропнутых таблиц и ошибок в коде.
