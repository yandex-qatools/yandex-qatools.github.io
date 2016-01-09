---
layout: post
title:  "Внешний вид в Jenkins - Jelly vs Groovy"
date:   2016-01-10 00:00:00 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [jenkins, jenkins plugin]
comments: true
published: true
---

## В общем о темплейтах

По написанию сложного UI для Jenkins информации не очень много. Обычно это сводится к советам "смотри как сделано у других".
Я далеко уходить от этой концепции не буду, остановлюсь лишь на теме Jelly *vs* Groovy (и быть может немножко о чистом JS). 
Сюда же накидаю ссылок, дабы самому не потерять.

### Общая информация

Вопрос выбора формата шаблонов для плагинов неоднозначен, потому что зачастую это дело вкуса, 
когда задача заключается в отрисовке простой формочки. Groovy темплейты в конечном итоге всё равно превращаются в jelly и только потом 
обрабатываются. Из этого следует один простой вывод - если вы только начинаете писать свои шаблоны - используйте jelly. 
Дальше при наборе опыта, вы сможете легко сконвертировать его в groovy.

Информацию о базовых тегах, доступных в jenkins стоит искать здесь:

- [Jelly Taglib Reference](https://jenkins-ci.org/maven-site/jenkins-core/jelly-taglib-ref.html)
- [Stapler Taglib Reference](http://stapler.kohsuke.org/jelly-taglib-ref.html)
- [Commons Jelly](http://commons.apache.org/proper/commons-jelly/tags.html)

Все что верно для Jelly, должно работать точно так же и для groovy. Просто в другом синтаксисе. 

### Когда использовать Groovy вместо Jelly

Было небольшое [обсуждение](https://groups.google.com/forum/#!topic/jenkinsci-dev/n4Xw9pq2fl8) этой темы в мейл-листе. 
От себя добавлю, что сейчас я предпочитаю для новых страниц использовать groovy. В основном потому что он более лаконичен и 
в случае со сложной логикой, его можно дебажить. Вот прям поставить брейкпоинт в темплейте и посмотреть что внутри.
С jelly такого не выйдет.

### Как начать переезд на groovy

Для начала стоит поглядеть на чужие темплейты поискав среди плагинов `config.groovy`, `global.groovy` в их ресурсах. 
Много примеров можно найти в [GitHub Plugin](https://github.com/jenkinsci/github-plugin/tree/master/src/main/resources).

У каждого groovy темплейта есть пакет, который соответствует полному классовому имени:

{% highlight java %}
package org.jenkinsci.plugins.github.config.GitHubTokenCredentialsCreator
{% endhighlight %}

И секция с необходимыми неймспейсами:

{% highlight java %}
def f = namespace(lib.FormTagLib);
def c = namespace(lib.CredentialsTagLib)
{% endhighlight %}

Они аналогичны тем что содержатся в теге jelly, за исключением того что ссылаются на предподготовленный dsl:

{% highlight xml %}
<j:jelly xmlns:j="jelly:core" xmlns:f="/lib/form">
{% endhighlight %}

Дальше, алгоритм довольно прост - берем тег, удаляем `<>`, добавляем после `{}`.

Т.е. например был код:
{% highlight xml %}
<f:entry title="${ %Credentials}" field="credentialsId">
        <f:select />
</f:entry>
{% endhighlight %}

Станет:

{% highlight groovy %}
f.entry(title: _('Credentials'), field: 'credentialsId') {
    c.select()
}
{% endhighlight %}

#### Особенности

Конструкция `_('Credentials')` значит тоже что и `${ %Credentials}` - локализуемость. В груви для обычных строк принято 
использовать одинарные кавычки. Ибо двойные - это особые GStrings, которые чуть медленнее и прожорливее. Зато внутри двойных 
кавычек автоматически раскрываются переменные в формате `"переменная ${var} - раскроется"`.

В груви `it` зарезервировано для итератора в замыканиях. Поэтому вместо него используется `my` или `instance`. 

Так же, стоит учесть, что в jelly поля со сложным путем, вроде `instance.config.var` при `null` на любом участке примут пустое значение.
В груви темплейтах такое стоит обрабатывать отдельно и самостоятельно - `instance?.config?.var`. 

При этом в момент первого отображения конфига, `instance` будет `null`, 
вне зависимости от того, что стоит в поле соответствующего класса как дефолт.

### Откуда берутся всякие lib.FormTagLib

Их генерирует из jelly файлов [maven-hpi-plugin](https://github.com/jenkinsci/maven-hpi-plugin/blob/master/src/main/java/org/jenkinsci/maven/plugins/hpi/TagLibInterfaceGeneratorMojo.java).
Пример вызова есть в credentials-plugin [pom.xml](https://github.com/jenkinsci/credentials-plugin/blob/master/pom.xml#L179-L195)

### А что про JS?

В Jenkins 2.0 готовится новый внешний вид, но концепция на которой он разрабатывается доступна уже сегодня. 
Подробнее узнать о ней можно в [jenkinsci/js-modules](https://github.com/jenkinsci/js-modules), а посмотреть туториал в 
[jenkinsci/js-samples](https://github.com/jenkinsci/js-samples). Концепция предлагает модульную разработку js внешнего вида.

На самом деле концепция не изобретает ничего нового. И использовать ее можно было уже давно. В Action можно было подключить абсолютно 
любой JS. Добавив веб-методов и экспорта для организации подобия REST-апи, можно сделать вполне современный и отзывчивый интерфейс.

Пример применения новой концепции доступен в [cloudbees/pipeline-editor-plugin](https://github.com/cloudbees/pipeline-editor-plugin)
 
