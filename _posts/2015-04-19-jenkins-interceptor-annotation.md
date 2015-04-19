---
layout: post
title:  "Пишем Jenkins Plugin: @InterceptorAnnotation"
date:   2015-04-19 22:30:00 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [jenkins, jenkins plugin]
comments: true
published: true
---

## Что это за аннотация

Веб-методы можно обрамлять специальными интерсепторами, что позволяет добавлять какой-то пред или пост-процессинг.
Наиболее доступным примером является требование чтобы запрос был POST запросом. 
[Интерсептор](http://stapler.kohsuke.org/apidocs/?Interceptor) для такой обработки 
поставляется вместе с аннотацией `@RequirePOST`. Вот она как раз и помечена указанной 
в заголовке [`@InterceptorAnnotation`](http://stapler.kohsuke.org/apidocs/?InterceptorAnnotation).

### Для чего можем использовать ее мы

Как упоминал [ранее](../jenkins-injected-parameter), интерсепторы вызываются уже после того как все аргументы 
проинициализированы значениями. Значит логично проверить запрос и аргументы на соответствие требованиям. Благо 
теперь мы не ограничены в управлении возвращаемым ответом.

### Пример применения

Чтобы в нашем веб-методе просто взять и начать работать с объектом стоит проверить что это вообще был подходящий запрос, 
и проверить аргумент на пустоту. Делаем все аналогично аннотации для инжекта параметров.

#### 1. Заводим аннотацию с говорящим названием и переопределяем метод интерсептора

{% highlight java %}
@Retention(RUNTIME)
@Target({METHOD, FIELD})
@InterceptorAnnotation(RequirePostWithPayload.Processor.class)
public @interface RequirePostWithPayload {
    class Processor extends Interceptor { }
}
{% endhighlight %}

Аннотацией `@RequirePostWithPayload` мы потом будем помечать веб-метод.

#### 2. Реализуем метод интерсептора

Хендлер должен переопределять один метод для успешной работы - 
{% highlight java %}
@Override
public Object invoke(
    StaplerRequest request, 
    StaplerResponse response, 
    Object instance, 
    Object[] arguments) throws IllegalAccessException, InvocationTargetException {

    // В случае отсутствия модификаций просто возвращаем результат выполнения
    return target.invoke(request, response, instance, arguments);
}
{% endhighlight %}

Я просто проверил что в аргументах есть хоть что-то и нужного типа. Обратите внимание, как легко это сделать матчерами!

{% highlight java %}
if (either(everyItem(nullValue(ServiceWebhook.class)))
                    .or(not(hasItem(isA(ServiceWebhook.class)))).matches(newArrayList(arguments))) {
    throw new InvocationTargetException(error(SC_BAD_REQUEST, "No valid payload in hook"));
}
{% endhighlight %}

### PROFIT!

Теперь логику предпроверки мы тоже можем протестировать отдельно, а в веб-методе не заниматься никакими проверками, 
оставляя код чистым и логичным.
