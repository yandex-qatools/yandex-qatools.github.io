---
layout: post
title:  "Пишем Jenkins Plugin: @InjectedParameter"
date:   2015-04-19 22:00:00 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [jenkins, jenkins plugin]
comments: true
published: true
---

## Для чего эта аннотация

Веб-методы позволяют инжектить параметры. Параметры могут быть извлечены из запроса, следуя определенной логике. 
Логика содержится в определенном annotation handler, а определяет какой взять - собственная аннотация для параметра 
при помощи [`@InjectedParameter`](http://stapler.kohsuke.org/apidocs/?InjectedParameter).
Пометив аргумент метода своей аннотацией, содержащей отсылку к определенному 
классу [`AnnotationHandler`](http://stapler.kohsuke.org/apidocs/?AnnotationHandler)'a, 
мы заставим при инжекте значений сперва вызвать хендлер для этого параметра. Таким образом уже реализованы 
аннотации `@Header` и `@QueryParameter`.

#### Что это дает

Мы можем описать всю логику парсинга запроса в виде одного хендлера, избавив сам веб-метод от этого кода. Что позволит 
нам тестировать и поддерживать это отдельно друг от друга. Чем это хорошо, надеюсь, объяснять не нужно. 

### Пример применения

Мне это понадобилось чтобы организовать парсинг вебхука от одного из сервисов в своем плагине.
Что было дано: `UnprotectedRootAction` с методом `doIndex` + сервис, кидающий POST запросом json. 
Что хотелось: получить в веб-методе готовый к работе объект, десериализованный из json.

Действуем:

#### 1. Заводим аннотацию с говорящим названием и хендлером

{% highlight java %}
@Retention(RUNTIME) // Взаимодействие с аннотацией происходит в рантайме
@Target(PARAMETER)  // Этой аннотацией можно помечать только параметры
@Documented         // Будет отражена в джавадоке
@InjectedParameter(ServicePayload.PayloadHandler.class) // Именно отсюда возьмется нужный класс хендлера
public @interface ServicePayload {
    class PayloadHandler extends AnnotationHandler<ServicePayload> { }
}
{% endhighlight %}

Аннотацией `@ServicePayload` мы потом будем помечать параметры в веб-методе.

#### 2. Реализуем метод хендлера

Хендлер должен переопределять один метод для успешной работы - 
{% highlight java %}
@Override
public Object parse(
    StaplerRequest request, // Отсюда мы возьмем данные
    ServicePayload a,     // На вход поступает и аннотация, откуда можно взять дополнительную метаинформацию
    Class type,             // Тип параметра в веб-методе
    String parameterName    // Имя параметра в веб-методе
) throws ServletException { }
{% endhighlight %}

В простом случае нужно прочитать данные из запроса и десериализовать. Я использовал Gson для этих целей, ибо 
мне была нужна своя логика десериализации, уже готовая на gson (хотя обычно хватает и возможностей stapler). 

{% highlight java %}
@Override
public Object parse(StaplerRequest request, ServicePayload a, Class type, String parameterName) throws ServletException {
     LOGGER.log(INFO, "[Service Plugin] Webhook received with method {0}", request.getMethod());

     String payload = trimToEmpty(extractContent(request));
     LOGGER.log(FINE, "Payload content -> {0}", payload);
     return new Gson().fromJson(payload, ServiceWebhook.class);
}

private String extractContent(StaplerRequest request) {
     try {
        return IOUtils.toString(request.getReader());
     } catch (IOException e) {
        return null;
     }
}
{% endhighlight %}

Остается только слегка предостеречься от неожиданностей и добавить проверок. 

> Забегая вперед, скажу, что этот код будет вызван ДО интерсепторов вводимых в бой  через `@InterceptorAnnotation`. 
При этом, здесь мы очень ограничены в ответах - любое исключение будет обернуто в 500ку и выплюнуто трейсом прямо в Jenkins.
Я считаю что в этом методе не стоит кидать исключений без крайней нужды, а валидировать полученный результат где-то еще.
Так же это вызывает необходимость логировать некоторые факты именно здесь, ибо в случае неожиданностей, процесс прервется и 
сам нигде не залогируется.

#### 3. Страхуемся от неожиданностей 

Во-первых, не стоит верить программистам (а особенно себе). Поэтому важно валидировать не только пользовательский ввод, но 
и те места где можно легко ошибиться самому - тут, например, это тип содержимого. Во-вторых, нам может придти от пользователя 
совсем не то что мы ждем. Это тоже стоит проверять.

Со всеми модификациями код метода становится таким: 

{% highlight java %}
@Override
public Object parse(StaplerRequest request, ServicePayload a, Class type, String parameterName) throws ServletException {
     isTrue(ServiceWebhook.class.isAssignableFrom(type),
            "Parameter '%s' should has type %s, not %s", parameterName,
            ServiceWebhook.class.getName(),
            type.getName()
     );

     LOGGER.log(INFO, "[Service Plugin] Webhook received with method {0}", request.getMethod());

     String payload = trimToEmpty(extractContent(request));
     LOGGER.log(FINE, "Payload content -> {0}", payload);
     
     JsonElement json = new JsonParser().parse(payload);
     
     if (!json.isJsonObject()) {
         return null;
     }
                 
     return new Gson().fromJson(json, ServiceWebhook.class);
}
{% endhighlight %}

#### 4. Тестируемся!

Я написал тест на то что валидный хук действительно верно десериализуется, а дальше на исключение по вине программиста, 
разные виды json (массив, пустота, невалидный), и один тест на исключение во время чтения данных из запроса.

## Итог

В итоге мы получаем симпатичный метод с приятной сигнатурой в коде экшена.
{% highlight java %}
public HttpResponse doIndex(@ServicePayload ServiceWebhook payload) throws ServletException { }
{% endhighlight %}

Вместо не очень очевидного 

{% highlight java %}
public HttpResponse doIndex(StaplerRequest request) throws ServletException { }
{% endhighlight %}



