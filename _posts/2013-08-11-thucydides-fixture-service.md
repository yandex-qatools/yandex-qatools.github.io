---
layout: post
title:  "Thucydides Fixture Service"
date:   2013-08-11 22:22:04
tags: [thucydides]
published: true
---

#  ...или как подобраться к capabilities браузера

## Предыстория

Все началось с необходимости использовать в thucydides тестах прокси сервера, который инициализируется вместе с началом
теста. Так как в thucydides нет возможности управлять напрямую процессом инициации браузера, пришлось искать обходные пути.

## Обходной путь №1 (неверный)

Начал эксперименты я с довольно старой версией - кажется это была *0.9.88*. У объекта `Pages` есть метод `setDriver(WebDriver)`.

Например, в тесте, в момент, когда проинициализирован объект  `Pages`, делаем:

{% highlight java %}
Proxy proxy = server.seleniumProxy();
DesiredCapabilities capabilities = DesiredCapabilities.firefox();
capabilities.setCapability(CapabilityType.PROXY, proxy);
WebDriver webDriver = BrowserFactory.webdriver(capabilities);
pages.setDriver(webDriver);
{% endhighlight %}

после этого все степы получают вебдрайвер из этих `pages`, который оказывается нужным вам.

Только тут есть ряд проблем:

* Сусид не управляет этим драйвером, закрывать его придется самостоятельно
* Сусид не управляет этим драйвером, поэтому рестартануть его не сможет в случае чего
* Сусид ничего не знает про этот драйвер (и знать не хочет), поэтому скриншотов не будет :(

Можно пойти дальше, и зарегистрировать в **EventBus** свой  **BaseStepListner**, который будет делать скриншоты и сохранять
в указанную папку. Но мне так и не хватило терпения выяснить, куда девается информация об этих скриншотах в момент
сериализации кейса в xml и построения html.

## Необходной путь №1

Замучавшись бродить кругами, я решил изучить механизм получения нового вебдрайвера на свежей версии (ей оказалась *0.9.203*).
И, возможно, сделать его чуть более доступным и изменяемым. В процессе перебора исходников, внезапно был найден интерфейс
`net.thucydides.core.fixtureservices.FixtureService`

{% highlight java %}
public interface FixtureService {
    //Этот метод вызывается до начала тесткейсов
    void setup() throws FixtureException;

    //Этот метод вызывается после окончания всех тесткейсов
    void shutdown() throws FixtureException;

    //Этот метод вызывается при каждой инициализации драйвера
    void addCapabilitiesTo(DesiredCapabilities capabilities);
}
{% endhighlight %}

Из официальной документации было обнаружено только несколько строчек комментариев

> Load any implementations of the FixtureService class declared on the classpath.
> FixtureService implementations must be declared in a file called net.thucydides.core.fixtureservices.FixtureService
> in the META-INF/services directory somewhere on the classpath.

Означает это следующее:

* Реализуем интерфейс.

Например, нам нужно динамически перед стартом теста менять переменные окружения thucydides

{% highlight java %}
@Override
public void setup() throws FixtureException {
   //Получаем переменные окружения, с которыми работает thucyd
   EnvironmentVariables env = Injectors.getInjector()
                                    .getInstance(EnvironmentVariables.class);

   //Достаем значение нужной переменной
   String name = ThucydidesSystemProperty.REMOTE_DRIVER.from(env, "firefox");

   String url = getRemoteUrlBy(name);

   //Устанавливаем нужное значение
   env.setProperty(ThucydidesSystemProperty.REMOTE_URL.getPropertyName(), url);
}
{% endhighlight %}

* Создаем в classpath папку `META-INF`, а в ней `services` с файлом `net.thucydides.core.fixtureservices.FixtureService`

{% highlight bash %}
resources/
└── META-INF/
     └── services/
           └── net.thucydides.core.fixtureservices.FixtureService
{% endhighlight %}

Содержимое файла - набор названий классов, реализующих нужный интерфейс:

{% highlight bash %}
ru.yandex.qatools.thucydides.grid.GridBrowserLaunch
ru.yandex.qatools.thucydides.proxy.ProxyFixture
{% endhighlight %}

Методы каждого из них будут последовательно выполнены.

##  Известные проблемы

1. Никаких полей для передачи информации между методами.
Объект инстанцируется каждый раз, когда требуется вызов его метода. Мне не удалось передать между методами
никакой информации не используя сторонних хранилищ.

2. Метод `void addCapabilitiesTo(DesiredCapabilities capabilities);` вызывается 2 раза при использовании RemoteWebdriver'a.

### Что можно сделать при помощи этой технологии

Фактически, это подобие рул в JUnit. Реализацией этого интерфейса, удалось легко реализовать получение вебдрайвера с грида,
с балансировкой на клиенте. Работу прокси-сервера `net.lightbody.bmp.proxy` (он же browsermob-proxy). Быть может есть еще
задачи, подсильные этому механизму.