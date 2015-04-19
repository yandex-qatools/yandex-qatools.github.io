---
layout: post
title:  "Пишем Jenkins Plugin: аннотация @DataBoundSetter"
date:   2015-04-11 14:12:22 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [jenkins, jenkins plugin]
comments: true
published: true
---

## Альтернатива @DataBoundConstructor

Намедни рассматривая пулл-реквесты в плагины для дженкинса, обнаружил интересную аннотацию - [`@DataBoundSetter`](http://stapler.kohsuke.org/apidocs/org/kohsuke/stapler/DataBoundSetter.html). 
Не раз испытывая боль, добавляя десятый параметр в конструктор с 
аннотацией [`@DataBoundConstructor`](http://stapler.kohsuke.org/apidocs/org/kohsuke/stapler/DataBoundConstructor.html), 
поискал что это за зверь.

### Что говорит Kohsuke Kawaguchi

Оригинал сообщения можно найти в [гугл-группах](https://groups.google.com/forum/#!msg/jenkinsci-dev/58-DEvuJZWI/5QrxBZRFJ6IJ). 
В сообщении говорится о новой аннотации, которая позволит избавиться от громадных конструкторов с перечислением 
всех-всех необходимых пропертей. Аннотация доступна начиная с дженкинса **1.535**, есть 
[джавадок на сайте степлера](http://stapler.kohsuke.org/apidocs/org/kohsuke/stapler/DataBoundSetter.html)

#### Пример применения

Раньше приходилось делать так: 
{% highlight java %}
class Foo {
   int a,b,c,d;
      
   @DataBoundConstructor
   public Foo(int a, int b, int c, int d) {
        this.a = a;
        this.b = b;
        this.c = c;
        this.d = d;
    }
}
{% endhighlight %}     


Теперь можно закинуть только несколько параметров в конструктор, 
а для остальных параметров пометить аннотацией сеттер или само поле. 
При этом конструктор может остаться вообще без параметров. 

> Важно заметить, что сеттеры должны начинаться с префикса `set` и содержать имя поля и быть публичными. 
Поля же могут быть с любым модификатором доступа, главное чтобы совпадали с именем проперти

{% highlight java %}
class Foo {
    int a;
    int b;
    
    @DataBoundSetter
    private int c;
    
    @DataBoundSetter
    private int d;
      
    @DataBoundConstructor
    public Foo() {}

    @DataBoundSetter
    public void setA(int a) { this.a = a; }
      
    @DataBoundSetter
    public void setC(int b) { this.b = b; }
}
{% endhighlight %}