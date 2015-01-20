---
layout: post
title:  "Jasmine & Maven: запуск тестов для js плагином"
date:   2015-01-19 17:12:22 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [javascript]
tags: [jasmine, maven-plugin, jasmine-maven-plugin]
comments: true
published: true
---

## Зачем Maven?

Наверняка многие спросят - зачем мне использовать мавен, когда есть *npm*? Ответ прост - лень :) Это, наверное, первоочередная причина. 
Поэтому лучшим выходом будет использовать конечно специализированные инструменты. Но если вы адепт Maven и вам проще подкинуть 
пару xml-тегов в pom.xml, то читайте дальше.

Если подходить к вопросу более серьезно, то у такого решения есть ряд стоящих плюсов.
Предположим, вы пишете на java, а значит, скорее всего используете maven. 
И тут вдруг появляется необходимость сделать морду на js. Для определённости примера пусть это будет морда на AngularJS.

В этом случае нужно либо ставить все окружение с нативными инструментами js, либо просто настроить POM-файлик. Если морда ещё 
и живет в отдельном модуле, то весь процесс юнит-тестирования и релиза становится унифицированным для всего проекта. Конкретно - 
достаточно просто сделать `mvn test` и подождать отчета.

Итак имеем:

- Простая настройка для людей, знакомых со сборкой проекта maven'ом.
- Единый способ запуска тестов и релиза модуля в рамках большого java-проекта.

## Тесты для AngularJS

Я проделывал это для ангуляра, поэтому рассказывать буду про него. Для остальных библиотек и фреймворков думаю отличий не 
слишком много. Нам потребуется:

- [jasmine-maven-plugin](https://github.com/searls/jasmine-maven-plugin)
- [angular.js](http://yastatic.net/angularjs/1.2.23/angular.js)
- [angular-mocks.js](http://yastatic.net/angularjs/1.2.23/angular-mocks.js)

Сами тесты стоит положить в папочку `test` там, где это удобно (обычно кладут на одном уровне с js-кодом). Относительный путь 
от pom-файла потом нужно будет указать в конфиге плагина в теге `jsTestSrcDir`.

Все наборы тестов на одну тему в терминах [Jasmine](http://jasmine.github.io/) называются `spec`. По правилам, каждый файл 
с определенной спекой должен называться в формате `someName.spec.js` или `someNameSpec.js`. Мне больше нравится первый. 

На содержимом останавливаться не будем (это тема для отдельной статьи), просто добавим заглушку:

{% highlight javascript %}
describe("test", function () {
    //можно вложить сьют

    beforeEach(module('my.own.module.name'));

    it("should be ok", function () {
        expect(true).toBe(true);
    });
});
{% endhighlight %}

## Настройка плагина jasmine-maven-plugin 

Всё как обычно: в *build/plugins* секции добавляем плагин:

{% highlight xml %}
<plugin>
   <groupId>com.github.searls</groupId>
   <artifactId>jasmine-maven-plugin</artifactId>
   <version>${jasmine.plugin.version}</version>
   <extensions>true</extensions>
   <executions>
        <execution>
            <id>unit-test-running</id>
            <goals>
                <!-- запустить mojo "test" (есть еще bdd) -->
                <goal>test</goal>
            </goals>
            <!-- запускать на фазе `mvn test` -->
            <phase>test</phase>
        </execution>
   </executions>
   <configuration>
        <!-- где находится корень скриптов приложения -->
        <jsSrcDir>assets/js</jsSrcDir>
        <!-- где находится корень тестовых спек -->
        <jsTestSrcDir>assets/test</jsTestSrcDir>
        <!-- куда выгружать результат в surefire формате -->
        <jasmineTargetDir>${project.build.directory}/surefire-reports</jasmineTargetDir>
        <!-- инстанс какого webdriver поднимать (по умолчанию HtmlUnit, который работает странно) -->
        <webDriverClassName>org.openqa.selenium.phantomjs.PhantomJSDriver</webDriverClassName>
        <!-- эти скрипты необходимо загрузить до выполнения тестов -->
        <preloadSources>
            <!-- возможны как относительные пути, так и полные урлы -->
            <source>http://yastatic.net/angularjs/1.2.23/angular.js</source>
            <source>http://yastatic.net/angularjs/1.2.23/angular-mocks.js</source>
        </preloadSources>
    </configuration>
</plugin>
{% endhighlight %}

На момент написания статьи использовал [1.3.1.2](https://github.com/searls/jasmine-maven-plugin/releases/tag/jasmine-maven-plugin-1.3.1.2) 

Проверям: `mvn clean test`

Вывод должен быть примерно следующим: 

{% highlight text %}
[INFO] --- jasmine-maven-plugin:1.3.1.2:test (unit-test-running) @ aqua-run ---
2015-01-19 19:57:57.172:INFO:oejs.Server:jetty-8.1.10.v20130312
2015-01-19 19:57:57.237:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:61417
[INFO] Executing Jasmine Specs
PhantomJS is launching GhostDriver...
[INFO  - 2015-01-19T15:57:58.695Z] GhostDriver - Main - running on port 37131
[INFO  - 2015-01-19T15:57:59.450Z] Session [f0d6dd40-9ff3-11e4-8f16-df2c937cf604] - page.settings - {"XSSAuditingEnabled":false,"javascriptCanCloseWindows":true,"javascriptCanOpenWindows":true,"javascriptEnabled":true,"loadImages":true,"localToRemoteUrlAccessEnabled":false,"userAgent":"Mozilla/5.0 (Macintosh; PPC Mac OS X) AppleWebKit/534.34 (KHTML, like Gecko) PhantomJS/1.9.7 Safari/534.34","webSecurityEnabled":true}
[INFO  - 2015-01-19T15:57:59.450Z] Session [f0d6dd40-9ff3-11e4-8f16-df2c937cf604] - page.customHeaders:  - {}
[INFO  - 2015-01-19T15:57:59.450Z] Session [f0d6dd40-9ff3-11e4-8f16-df2c937cf604] - Session.negotiatedCapabilities - {"browserName":"phantomjs","version":"1.9.7","driverName":"ghostdriver","driverVersion":"1.1.0","platform":"mac-unknown-64bit","javascriptEnabled":true,"takesScreenshot":true,"handlesAlerts":false,"databaseEnabled":false,"locationContextEnabled":false,"applicationCacheEnabled":false,"browserConnectionEnabled":false,"cssSelectorsEnabled":true,"webStorageEnabled":false,"rotatable":false,"acceptSslCerts":false,"nativeEvents":true,"proxy":{"proxyType":"direct"}}
[INFO  - 2015-01-19T15:57:59.450Z] SessionManagerReqHand - _postNewSessionCommand - New Session Created: f0d6dd40-9ff3-11e4-8f16-df2c937cf604
[INFO] 
-------------------------------------------------------
 J A S M I N E   S P E C S
-------------------------------------------------------
[INFO] 
test
  should be ok

Results: 1 specs, 0 failures

2015-01-19 19:58:00.670:INFO:oejsh.ContextHandler:stopped o.e.j.s.h.ContextHandler{/,file:/Users/lanwen/git/tools-files/tools/aquarun/}
2015-01-19 19:58:00.670:INFO:oejsh.ContextHandler:stopped o.e.j.s.h.ContextHandler{/spec,file:/Users/lanwen/git/tools-files/tools/aquarun/}
2015-01-19 19:58:00.670:INFO:oejsh.ContextHandler:stopped o.e.j.s.h.ContextHandler{/src,file:/Users/lanwen/git/tools-files/tools/aquarun/}
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
{% endhighlight %}

## BDD & плагин

Если сделать `mvn jasmine:bdd`, запустим серверок, который можно открыть в отдельной владке браузера и с рефрешем получать 
мгновенный результат прохождения тестов:

{% highlight text %}
[INFO] --- jasmine-maven-plugin:1.3.1.2:bdd (default-cli) @ aqua-run ---
2015-01-19 20:04:00.094:INFO:oejs.Server:jetty-8.1.10.v20130312
2015-01-19 20:04:00.149:INFO:oejs.AbstractConnector:Started SelectChannelConnector@0.0.0.0:8234
[INFO] 

Server started--it's time to spec some JavaScript! You can run your specs as you develop by visiting this URL in a web browser: 

  http://localhost:8234

The server will monitor these two directories for scripts that you add, remove, and change:

  source directory: assets/js

  spec directory: assets/test

Just leave this process running as you test-drive your code, refreshing your browser window to re-run your specs. You can kill the server with Ctrl-C when you're done.
{% endhighlight %}


Зайдя на `http://localhost:8234` увидим [аналогичную страничку](http://jasmine.github.io/1.3/introduction.html) (внизу)


