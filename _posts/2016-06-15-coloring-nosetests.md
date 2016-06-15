---
layout: post
title:  "Раскрашиваем nose тесты"
date:   2016-06-15 00:00:00 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [python]
tags: [nosetests, python-testing]
comments: true
published: true
---

# Облегчаем восприятие от тестов

Оригинальная проблема простая - имеем портянку plain текста после пары фейлов. Как сделать красивше?

## YANC

[Python package](https://pypi.python.org/pypi/yanc)

Не особо понравился. Слишком мало чего вразумительного раскрашивает. Никаких опций. Вызывается как `--with-yanc`

![yanc](http://blog.qatools.ru/images/2016-06-15_21-13-40.png)

## Rednose

[Python package](https://pypi.python.org/pypi/rednose)

Выглядит гораздо более вразумительно. Умеет выплевывать фейлы сразу же, 
умеет сам определять что раскрашивать не нужно (например когда льется всё в файл). Вызывается в минимуме как `--rednose`

![rednose](http://blog.qatools.ru/images/2016-06-15_21-24-01.png)
