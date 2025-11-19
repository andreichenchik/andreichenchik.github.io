---
title: Cloudflare-прокси для моего Dokku сервера
date: 2025-08-17T10:00:00+02:00
draft: false
description: 'Как я прикрутил бесплатный прокси Cloudflare к своему минимальному Dokku-серверу, получил HTTPS и защитил его фаерволом.'
summary: 'Как спрятать Dokku-приложение за Cloudflare: перенести домен, включить прокси, выдать origin-сертификат и закрыть сервер фаерволом.'
tags: ["infra", "cloudflare", "dokku", "hetzner"]
ShowToc: true
TocOpen: false
cover:
  image: cover.png
---

Продолжаю историю, начатую с [минимальным Dokku-сервером](/ru/posts/minimal-dokku-setup/). Это бонусный кусок, который я вынес отдельно, чтобы не раздувать основную статью.

Напомню: приложение `helloworld` уже [доступно](https://helloworld.chenchik.me), сервер работает исправно. Теперь хочется добавить запас по производительности и безопасности, чтобы не упираться в железо. Для этого я ставлю **Cloudflare** между пользователем и сервером: сервис бесплатный, даёт кэш и фильтрует большую часть спам-запросов до того, как они попадут на VPS.

## Что потребуется

- зарегистрированный домен, готовый к переносу в [Cloudflare](https://cloudflare.com);
- учётная запись Cloudflare;
- сервер с установленным Dokku и приложением `helloworld`;
- доступ к Hetzner Firewall (или к фаерволу вашего провайдера), чтобы ограничить трафик IP-адресами Cloudflare.

## Пошаговая инструкция

### 1. Переносим домен в Cloudflare

Первая задача — передать Cloudflare управление DNS. Это один из лучших DNS-сервисов: он быстрый и бесплатный, поэтому даже сам перенос уже приносит пользу. В панели Cloudflare создаю новую зону и следую мастеру. У каждого регистратора свои шаги, поэтому проще заглянуть в официальный гайд: https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/ После смены NS-записей остаётся дождаться, пока Cloudflare подтвердит зону.

### 2. Включаем прокси для DNS-записей

В доменной зоне уже лежат A-записи на мой Hetzner IP. По умолчанию Cloudflare просто отвечает на DNS-запросы (серая иконка). Мне нужен полноценный прокси, поэтому я включаю оранжевое облако (Proxied) у записи `helloworld`. Теперь весь HTTP/HTTPS-трафик идёт через Cloudflare, а бесплатные Edge-сертификаты включаются автоматически. На этом этапе приложение уже доступно по `https://helloworld.chenchik.me`.

### 3. Генерируем Origin Certificate

Такое подключение работает, но обновлять Let's Encrypt-сертификаты теперь неудобно, особенно если приложений несколько — Brian в деталях рассказал про эту проблему: https://bitkidd.dev/posts/use-cloudflare-ssl-certificates-with-dokku Чтобы не упираться в rate limit и не держать публичные сертификаты за Cloudflare, лучше выпустить **Origin Certificate**. Он защищает трафик между Cloudflare и сервером и действует до 15 лет. В панели захожу в **SSL/TLS → Origin Server** и нажимаю «Create Certificate»: сервис отдаёт приватный ключ и сертификат, сохраняю их как `pet.key` и `pet.crt`.

### 4. Доставляем сертификат в Dokku

Нужно перенести файлы на сервер и включить их в Dokku.

1. Подключаюсь к серверу: `ssh root@<ip>`.
2. Создаю файлы с сертификатом и ключом (`nano pet.crt`, `nano pet.key`) и вставляю выданные Cloudflare данные.
3. Упаковываю их, отключаю старый сертификат и добавляю новый:

   ```shell
   tar -cvf pet.tar pet.crt pet.key
   dokku letsencrypt:disable helloworld
   dokku certs:add helloworld < pet.tar
   dokku ps:rebuild helloworld
   ```

Команда `certs:add` перезапишет сертификат, а `ps:rebuild` пересоберёт контейнер с новыми TLS-настройками. После этого Cloudflare устанавливает TLS до своего Edge, а Edge общается с сервером также по TLS — цепочка полностью зашифрована.

Когда все заработало, удаляю больше не нужный плагин Let’s Encrypt:

```shell
sudo dokku plugin:uninstall letsencrypt
```

### 5. Закрываем сервер фаерволом

Пока любой может обойти Cloudflare и постучаться прямо на IP сервера. Чтобы этого не было, в Hetzner Cloud включаю **Firewall** и разрешаю входящие соединения на 80/443 только с IP-пулов Cloudflare: https://www.cloudflare.com/ips/ SSH оставляю лишь для собственных адресов. Так сервер «видит» интернет исключительно через Cloudflare. Catalin отлично показывает эту схему на примере Hetzner + Cloudflare: https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/

## Финальные проверки

Проверяю, что сертификат подтянулся:

```shell
curl -I https://helloworld.chenchik.me
```

`server: cloudflare` и `cf-cache-status: DYNAMIC` подтверждают, что запросы идут через прокси. При прямом обращении к IP порты 80/443 уже недоступны (файрвол).

Вся настройка занимает 30–60 минут. На выходе — больше производительности за счёт кэша, бесплатные HTTPS-сертификаты и защита от случайных нагрузок. Дальше буду добавлять функциональность для пет-проектов.

## Источники

- https://bitkidd.dev/posts/use-cloudflare-ssl-certificates-with-dokku
- https://spiffy.tech/dokku-with-lets-encrypt-behind-cloudflare
- https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/
- https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/
