---
layout: post
title:  "Store test accounts property-way"
date:   2013-08-25 22:22:22 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [property-loader]
tags: [property, gson, deserializer]
comments: true
published: true
---

# ...или расширенное применение собственных конвертеров в property-loader

## Зачем это?

Частенько возникают ситуации, когда нужно сохранить много текстовой информации, которая при этом обладает определенной структурой.
Примером такой информации может быть набор тестовых аккаунтов, которые могут использоваться для различных действий в ваших тестах.
В текстовом виде, в отдельном файле эту информацию очень легко поддерживать - легко подменить, легко отредактировать.
Одним из самых распространенных на сегодняшний день форматов для хранения текстового представления объектов является json -
его и будем использовать.

## Задача

Итак, ставим задачу - хранить в json готовые объекты, представляющие собой тестовые аккаунты в виде:


{% highlight javascript %}
{
    "default": {
        "login": "login1",
        "pwd": "pwd1"
    },

    //...
    "key1": {
        "login": "login2",
        "pwd": "pwd2"
    }
}
{% endhighlight %}

И получать к ним доступ из поля `Map<String, Acc> accounts;`


## Что нужно для реализации

Вооружимся [статьёй из мануала к property-loader](https://github.com/yandex-qatools/properties/blob/master/properties-loader/src/site/creation-custom-converter.ru.md),
[самой библиотекой](https://github.com/yandex-qatools/properties) (на текущий момент она имеет версию 1.4) и классом, представляющим единичный аккаунт:

{% highlight java %}
public class Acc {
    private String login;
    private String pwd;

    public Acc(String suid, String mdb) {
        this.login = suid;
        this.pwd = mdb;
    }

    public String getLogin() {
        return login;
    }

    public String getPwd() {
        return pwd;
    }
}
{% endhighlight %}

## Самое интересное

Весь механизм превращения json-файлика в набор аккаунтов нам обеспечит библиотека. Остается только прописать структуру

{% highlight java %}
@Resource.Classpath("accounts.properties")
public class AccountsProperties {

    private static AccountsProperties instance;

    // Ленивая инициализация + синглтон
    public static AccountsProperties props() {
        if (instance == null) {
            instance = new AccountsProperties();
        }
        return instance;
    }

    private AccountsProperties() {
        PropertyLoader.populate(this);
    }

    @Use(AccountsConverter.class)
    @Property("accounts.file")
    private Map<String, Acc> accounts;
}
{% endhighlight %}

И указать в `accounts.property` местонахождение нужного файлика. Пусть он тоже лежит в classpath

{% highlight java %}
accounts.file=accounts.json
{% endhighlight %}

### Наконец, сам конвертер:

{% highlight java %}
public class AccountsConverter implements Converter {
    @Override
    public Object convert(Class aClass, Object o) {
        if (!(o instanceof String)) {
            // в любой непонятной ситуации возвращаем пустую мапу,
            // аналогично можно действовать и
            // в случае проблем с чтением.
            return new HashMap<String, Acc>();
        }
        String path = (String) o;
        try {
            // Пользуемся commons-io для облегчения конвертации
            String json = IOUtils.toString(this.getClass().getClassLoader().getResourceAsStream(path));
            // Эта магическая строка преобразует наш json в мапу аккаунтов
            return new Gson().fromJson(json, new TypeToken<Map<String, Acc>>() {
            }.getType());
        } catch (IOException e) {
            throw new RuntimeException("Невозможно прочитать " + path, e);
        }
    }
}
{% endhighlight %}

## Как это использовать

Теперь в тестах можно получить аккаунт по ключу так:
{% highlight java %}
Acc account = props().account("key");
{% endhighlight %}

Если на один тестовый класс приходится один тестовый аккаунт, можно вполне сделать ключом имя класса. Тогда в `AccountsProperties`
стоит добавить геттеры такого рода:

{% highlight java %}

    public <T> Acc account(T bean) {
        return account(bean.getClass());
    }

    public Acc account(Class clazz) {
        return account(clazz.getSimpleName());
    }

    public Acc account(String name) {
        return accounts.containsKey(name) ? accounts.get(name) : accounts.get("default");
    }

{% endhighlight %}

и в тесте можно получать аккаунт так:
{% highlight java %}
Acc account = props().account(this);
{% endhighlight %}

## Дальнейшее развитие

Можно подставлять нужный файл аккаунтов в зависимости от окружения, от текущей ситуации, можно вывести хранение файла в
отдельное место, подправив класс конвертера. Можно аналогично хранить и любую другую информацию.
Например сериализовать начальные данные, а потом поднимать их таким образом.

По поводу того, как реализовать возможность выбора окружения, можно почитать
[в статье о property provider](https://github.com/yandex-qatools/properties/blob/master/properties-loader/src/site/creation-custom-property-provider.ru.md)
библиотеки.