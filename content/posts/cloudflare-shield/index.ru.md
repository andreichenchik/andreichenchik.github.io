---
title: Cloudflare-прокси для моего Dokku сервера
date: 2025-08-17T10:00:00+02:00
draft: true
description: 'Как я прикрутил бесплатный прокси Cloudflare к своему минимальному Dokku-серверу, получил HTTPS и защитил его фаерволом.'
summary: 'Рассказываю, как ускорить бэкенд на Dokku используя Cloudflare: перенести домен, выдать origin-сертификат, включить прокси и закрыть сервер фаерволом.'
tags: ["infra", "cloudflare", "dokku", "hetzner"]
ShowToc: true
TocOpen: false
cover:
  image: cover.png
---

Продолжаем историю с [минимальным Dokku-сервером](/ru/posts/minimal-dokku-setup/): Приложение `helloworld` уже доступно, сервер работает исправно, но можно немного ускорить его работу. Чтобы не упираться в железо, я добавляю **Cloudflare** между пользователем и сервером. Это бесплатно, даёт кэш, автоматичскую проверку сертификатов и фильтрует большую часть мусорных спам запросов до того, как запросы попадут ко мне на сервер.

### Что потребуется

- зарегистрированный домен, который можно перенести к [Cloudflare](https://cloudflare.com) (я купил его заранее для прошлой статьи);
- зарегистрироваться в Cloudflare;
- сервер с установленным Dokku и приложением `helloworld`;
- доступ к Hetzner Firewall (или к фаерволу вашего серверного провайдера), чтобы закрыть сервер от любого трафика, кроме cloudflare.


## Пошаговая инструкция 
### Переносим домен в Cloudflare

Задача — сделать Cloudflare авторитетной зоной для моего домена. В панели я создаю новую зону, указываю домен и следую мастеру переноса. У каждого регистратора свои шаги, поэтому удобнее смотреть официальные инструкции — Cloudflare собрал их [в отдельном гайде](https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/). После смены NS-записей жду пару часов: как только Cloudflare подтвердит зону, можно двигаться дальше.

### Включаем прокси для DNS-записей

В зоне уже лежат A-записи на мой Hetzner IP и поддомен `helloworld`. По умолчанию DNS работает «серой» и лишь отвечает на запросы. Мне нужен именно прокси, поэтому я включаю оранжевое облако (Proxied) у записи `helloworld`. Теперь весь HTTP/HTTPS-трафик идёт через Cloudflare: кеш, gzip, HTTP/2 и бесплатные Edge-сертификаты активируются автоматически. На этом этапе приложение уже доступно по `https://helloworld.chenchik.me`, но сертификат принадлежит Cloudflare, а между Cloudflare и сервером пока ездит обычный HTTP.

### Генерируем Origin Certificate

Чтобы исключить MitM между Cloudflare и моим Dokku, я выпускаю **Origin Certificate**. Это бесплатный сертификат, который доверяет только Cloudflare. В панели захожу в **SSL/TLS → Origin Server** и генерирую пару на 15 лет. Cloudflare выдаёт мне приватный ключ и сертификат — я сохраняю их как `cloudflare-origin.key` и `cloudflare-origin.pem`. Этим же этапам посвящён отличный разбор от Bitkidd: https://bitkidd.dev/posts/use-cloudflare-ssl-certificates-with-dokku

Почему именно origin-сертификат? С обычным Let's Encrypt я упрямся в rate limit, да и нет смысла тратить публичный сертификат на трафик, который остаётся внутри Cloudflare. Эту идею хорошо описывает защита Dokku за Cloudflare у Spiffy: https://spiffy.tech/dokku-with-lets-encrypt-behind-cloudflare

### Ставим сертификат в Dokku

Дальше нужно доставить файлы на сервер и включить их в Dokku:

```shell
scp cloudflare-origin.* root@<ip>:/root/
ssh root@<ip>

dokku certs:remove helloworld || true
dokku certs:add helloworld /root/cloudflare-origin.pem /root/cloudflare-origin.key
dokku ps:rebuild helloworld
```

Команда `certs:add` перезапишет сертификат, а `ps:rebuild` пересоздаст контейнер с новыми TLS-настройками. После этого Cloudflare устанавливает TLS до своего Edge, а Edge общается с сервером также по TLS — цепочка полностью зашифрована. Проверяю, что сертификат подтянулся, командой:

```shell
curl -I https://helloworld.chenchik.me
```

`server: cloudflare` и `cf-cache-status: DYNAMIC` подскажут, что трафик идёт через прокси.

### Закрываем сервер фаерволом

Пока любой может обойти Cloudflare и стучаться прямо в мой IP. Чтобы это остановить, в Hetzner Cloud включаю **Firewall** и разрешаю входящие соединения на 80/443 только с IP-пулов Cloudflare (их список лежит в панели и регулярно дополняется). Входящий SSH оставляю только со своего домашнего IP. Так сервер «видит» интернет исключительно через Cloudflare, а значит, на нём не нужно гонять Fail2ban и ловить лишние сканы. У Catalin есть наглядный разбор с Hetzner и Cloudflare, я опирался именно на него: https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/

### Финальные проверки

- `curl -I https://helloworld.chenchik.me` возвращает 200 и заголовки Cloudflare.
- При прямом обращении к IP порты 80/443 закрыты (firewall).
- В Cloudflare → Security → Events видно реальные запросы, а на сервер прилетает только проксированный трафик.

На всё уходит минут 20–30, а в ответ получаешь больше производительности за счёт кэша, бесплатные HTTPS-сертификаты и защиту от случайных нагрузок. Следующий шаг — подключить наблюдаемость и следить, как ведёт себя мой мини-бэкенд под Cloudflare.

## Источники

- https://bitkidd.dev/posts/use-cloudflare-ssl-certificates-with-dokku
- https://spiffy.tech/dokku-with-lets-encrypt-behind-cloudflare
- https://catalins.tech/selfhost-with-dokku-hetzner-cloudflare/
- https://developers.cloudflare.com/registrar/get-started/transfer-domain-to-cloudflare/
