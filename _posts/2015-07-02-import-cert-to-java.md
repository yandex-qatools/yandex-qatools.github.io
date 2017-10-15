---
layout: post
title:  "Как импортировать сертификат в java"
date:   2015-07-02 01:30:00 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [ssl]
tags: [ssl, certificate]
comments: true
published: true
---

## Шпаргалка на случай внезапных стектрейсов

Важно! На самом деле лучше цивилизованно поставить корневой сертификат и не заниматься этой фигней. 
Но если очень надо, то вот...

#### 1. Качаем сертификат хоста

`$ openssl s_client -connect $DOMAIN_NAME:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | tee host.crt`

(Можно на самом деле не писать в файлик, а работать со стандартным выводом на этом шаге)

На IPv6-only хосте получил сертификат только через

`echo -n | gnutls-cli --print-cert -p 443 somehost-with-ssl`

#### 2. Редактируем в файл host.crt (для openssl необязательно)

удаляем все, что вокруг: 
`-----BEGIN CERTIFICATE-----`
 и 
`-----END CERTIFICATE-----`

#### 3. Импортируем в хранилище
 
`$ sudo keytool -import -trustcacerts -alias somehost-with-ssl -file host.crt -keystore /usr/lib/jvm/java-6-oracle/jre/lib/security/cacerts`

Путь до JRE указываем правильный. Пароль от *keystore* по-умолчанию `changeit`
