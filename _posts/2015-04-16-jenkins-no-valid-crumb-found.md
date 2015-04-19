---
layout: post
title:  "Тестируем Jenkins Plugin: Crumb, CSRF"
date:   2015-04-15 22:12:22 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [jenkins, jenkins plugin, crumb, CSRF]
comments: true
published: true
---

## «No valid crumb was included in the request» при тестах UnprotectedRootAction

Опуская все подробности того что такое `UnprotectedRootAction` и с чем его едят, остановимся на конкретном вопросе. 
Создаем плагин с таким экшеном, а в тестах настроив рулу дженкинса делаем POST запрос. 
И получаем `Error 403 No valid crumb was included in the request`. Что в этом случае делать?

### Что говорит официальное wiki

В разделе [CSRF Protection](https://wiki.jenkins-ci.org/display/JENKINS/Remote+access+API) написано что нужно получить 
некий CSRF токен. И даже дают пример как его получить 
(`JENKINS_URL/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)` с выводом вроде `.crumb:1234abcd`)

#### Что с этим делать

Я догадался не сразу, а только покопав слегка исходники. Оказывается это `заголовок:значение`, либо `параметр:значение`.
В тестах значение всегда равно `test` 
(благодаря [TestCrumbIssuer](http://javadoc.jenkins-ci.org/org/jvnet/hudson/test/TestCrumbIssuer.html)).
Поэтому запрос на получение токена можно не делать, а сразу использовать это примерно так: 

{% highlight java %}
@ClassRule
public static JenkinsRule jenkins = new JenkinsRule();

@Test
public void shouldGet200OnRequestToUnprotectedAction() throws Exception {
  given() .baseUri(jenkins.getInstance().getRootUrl())
          .header(".crumb", "test")
          .content("some content")
          .post("url-of-unprotected-action");
}
{% endhighlight %}