#Механизм правил в JUnit

## JUnit

С фрэймворком модульного тестирования http://junit.org/, кажется, знакомы все. 
Как видно из его описания, он направлен на создание "повторяемых" (repeatable) тестов. Что это значит?
Значит тесты, кторые мы написали, придётся ещё и поддерживать.
Поэтому в основную очередь стоит рассматривать данный инструмент как набор удобных механизмов, 
которые направлены на то, чтобы облегчить написание и дальнейшую поддержку наших тестов.
В данной статье мы остановимся подробней на механизме правил (Rules, далее просто Рула). 


## Fixture

Test fixture — особое состояние данных необходимое для успешного выполнения теста.
Чтобы подготовить данные для каждого отдельного теста в классе необходимо воспользоваться 
специальными аннотациями @Before и @After. Методы с такими аннотациями будут выполнятся, соответственно, 
"до" и "после" каждого теста:

{% highlight java %}

public class SimpleTest {

    @Before
    public void before() {
        System.out.println("before");
    }
     
    @After
    public void after() {
        System.out.println("after");
    }
     
    @Test
    public void test() {
        System.out.println("test");
    }
        
    @Test
    public void test2() {
        System.out.println("test2");
    }
}

{% endhighlight %}

Результат выполнения будет следующим:
> before  

> test  

> after  

> before  

> test2  

> after


## Rules

### Custom

Допустим теперь мы хотим переиспользовать этот код (before и after) в других классах. 
Вместо того, чтобы копировать методы целиком или выносить их в отдельный базовый класс для каждогого нового
набора тестов, нужно воспользоваться Рулами.

Рула представляет из себя класс, реализующий интерфейс org.junit.rules.TestRule. Для того, чтобы создать новую Рулу нужно реализовать метод apply, возвращающий объект типа org.junit.runners.model.Statement.

Каждый индивидуальный вызов теста в JUnit представляет собой вызов метода evaluate объекта типа Statement. 
В Руле мы просто оборачиваем выполнение теста (base.evaluate()) своим кодом:

{% highlight java %}

public class SimpleRule implements TestRule {

    @Override
    public Statement apply(final Statement base, Description description) {
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                System.out.println("before");
                base.evaluate();
                System.out.println("after");
            }
        };
    }
}

{% endhighlight %}

Чтобы использовать новую Рулу необходимо подключить её как поле с аннотацией org.junit.Rule

{% highlight java %}

public class SimpleTest {

    @Rule
    public SimpleRule rule = new SimpleRule();
     
    @Test
    public void test() {
        System.out.println("test");
    }
        
    @Test
    public void test2() {
        System.out.println("test2");
    }
}

{% endhighlight %}

Результат:
> before  

> test  

> after  

> before  

> test2  

> after

### Advance

JUnit предлагает использовать или расширять свои рулы "из коробки". 
Самой популярной из них, как мне кажется, является org.junit.rules.ExternalResource;

#### ExternalResource
Данную рулу предполагается использовать в тех случаях, когда подготовленные для тестирования данные нобходимо очистить (освободить) после прохождения теста. Посмотрим как это реализовано:

{% highlight java %}
@Override
public void evaluate() throws Throwable {
    before();
    try {
        base.evaluate();
    } finally {
        after();
    }
}
{% endhighlight %}

Здесь я выделил только метод evaluate, чтобы показать сходство с нашим предыдущим примером. Методы
before() и after() предполагается реализовать самому. В них как раз и нужно описать управление своими данными. 

##### TemporaryFolder
Рула org.junit.rules.TemporaryFolder является частным случаем ExternalResource, и позволяет создавать файлы и папки, которые гарантированно удалятся после завершения теста. Пример использования с сайта производителя:

{% highlight java %}

@Rule
public TemporaryFolder folder = new TemporaryFolder();

@Test
public void testUsingTempFolder() throws IOException {
    File createdFile = folder.newFile("myfile.txt");
    File createdFolder = folder.newFolder("subfolder");
    // ...
}

{% endhighlight %}

#### Verifier
Класс org.junit.rules.Verifier также как и ExternalResource является базовым классом, в котором предполагается самому реализвать метод verify():

{% highlight java %}

@Override
public void evaluate() throws Throwable {
    base.evaluate();
    verify();
}

{% endhighlight %}

##### ErrorCollector 
В качестве примера использования Verifier рассмотрим рулу org.junit.rules.ErrorCollector, которая позволяет 
"продолжить выпопление теста после первой ошибки". Использование этой рулы позволит, нармер, собрать все ошибки произошедшие в тесте в одном отчёте.

{% highlight java %}

@Rule
public ErrorCollector collector= new ErrorCollector();

@Test
public void example() {
    collector.addError(new Throwable("что-то пошло не так"));
    collector.addError(new Throwable("опять что-то пошло не так"));
}

{% endhighlight %}

#### TestWatcher
Следущая базовая Рула org.junit.rules.TestWatcher призвана добавить немного свободы в наши тесты, так как (здесь я не буду привдить её реализацию) предоставляет возможность переопределить следующие методы:

> succeeded

> failed

> starting

> finished

Из названий методов можно догадаться, в какой момент они выполнятся, а именно: начало или конец теста, успешное или не очень завершение теста.

##### TestName
Простая для понимания Рула org.junit.rules.TestName является наследником TestWatcher и позволяет использовать имя метода внутри него самого: 

{% highlight java %}
private String name;

@Override
protected void starting(Description d) {
    name = d.getMethodName();
}

public String getMethodName() {
    return name;
}

{% endhighlight %}


#### ExpectedException
Рула org.junit.rules.ExpectedException предоставляет удобный способ проверки бросаемого в тесте исключения (Exception):

{% highlight java %}

@Rule
public ExpectedException thrown= ExpectedException.none();

@Test
public void throwsNullPointerExceptionWithMessage() {
    thrown.expect(NullPointerException.class);
    thrown.expectMessage("happened?");
    thrown.expectMessage(startsWith("What"));
    throw new NullPointerException("What happened?");
}

{% endhighlight %}

#### RuleChain
Очень полезная Рула org.junit.rules.RuleChain приходит на помощь, когда для теста необходимо вызвать несколько Рул в строгом порядке. Например подготовка данных стотоит из нескольких последовательных шагов, каждый из которых уже описан. Например:

{% highlight java %}

@Rule
public TestRule chain= RuleChain
                       .outerRule(new SimplePrintRule("outer rule"))
                       .around(new SimplePrintRule("middle rule"))
                       .around(new SimplePrintRule("inner rule"));

@Test
public void example() {
    assertTrue(true);
}

public class SimplePrintRule implements TestRule {
    private String name;

    public SimplePrintRule(String name) {
        this.name = name;
    }

    @Override
    public Statement apply(final Statement base, Description description) {
        return new Statement() {
            @Override
            public void evaluate() throws Throwable {
                System.out.println("starting " + name);
                base.evaluate();
                System.out.println("finished " + name);
            }
        };
    }
}

{% endhighlight %}

В результате получим ожидаемый порядок выполнения
> starting outer rule

> starting middle rule

> starting inner rule

> finished inner rule

> finished middle rule

> finished outer rule


##
Подробней про Рулы можно узнать из репозитория проекта на гитхабе https://github.com/junit-team/junit
