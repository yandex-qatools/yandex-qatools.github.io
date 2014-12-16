#Механизм правил в JUnit

## JUnit

С фрэймворком модульного тестирования http://junit.org/, кажется, знакомы все. 
Как видно из его описания, он направлен на создание "повторяемых" (repeatable) тестов. Что это значит?
Значит тесты, кторые мы написали, придётся ещё и поддерживать.
Поэтому в первую очередь стоит рассматривать данный фреймворк как набор удобных механизмов, 
которые направлены на то, чтобы облегчить написание и дальнейшую поддержку наших тестов.
В данной статье мы остановимся подробней на механизме правил (Rules, далее просто Рул). 


## Fixture

Test fixture — это особое состояние данных необходимое для успешного выполнения теста. 
Допустим, есть два или более теста, которые работают с одинаковым набором данных (fixture).
Чтобы подготовить эти данные для каждого отдельного теста в классе необходимо воспользоваться 
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

@Before и @After являются самыми простыми рулами, вшитыми в ядро JUnit. Наравне с ними существуют рулы @BeforeClass и @AfterClass, которые работают аналогичным образом, только вызываются для целого класса, а не для каждого метода:

{% highlight java %}

public class SimpleTest {

    @BeforeClass
    public static void before() {
        System.out.println("before");
    }
     
    @AfterClass
    public static void after() {
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

Получим следующий результат:
> before  

> test  

> test2  

> after

Заметим, что методы с @BeforeClass и @AfterClass должны быть статическими. 

## Rules
### Custom Rules
Допустим теперь мы хотим использовать предыдущий код (методы before и after) в других классах. 
Вместо того, чтобы копировать методы целиком или выносить их в отдельный базовый класс для каждогого нового
набора тестов, нужно создать свою собственную рулу.

Рула представляет из себя класс, реализующий интерфейс org.junit.rules.TestRule. Для того, чтобы создать новую рулу необходимо реализовать метод apply, возвращающий объект типа org.junit.runners.model.Statement.

Далее, каждый индивидуальный вызов теста в JUnit представляет собой вызов метода evaluate как раз этого объекта типа Statement. В руле мы просто оборачиваем выполнение теста (base.evaluate()) своим кодом:

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

Для использования рулы у себя необходимо подключить её как поле соответствующего класса с аннотацией org.junit.Rule или статическое поле с аннотацией org.junit.ClassRule, в зависимости от поставленной цели. При запуске теста JUnit будет ориентироваться только по этим аннотациям и рулы без них (если такие имеются) будут проигнорированы.  

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

### Base Rules

Прежде чем начать писать свои собственные рулы следует познакомиться с уже существующими. Фреймворк предлагает несколько готовых рул с удобными методами, которые можно использовать "из коробки".
Самой популярной из них, как мне кажется, является org.junit.rules.ExternalResource;

#### ExternalResource
Данную рулу предполагается использовать в тех случаях, когда подготовленные для тестирования данные нобходимо очистить (освободить) при любом исходе теста. Посмотрим как это реализовано:

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
before() и after() предполагается реализовать самому. В них и нужно описать управление своими данными. 

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

#### TestWatcher
Следущая базовая Рула org.junit.rules.TestWatcher не менее популярна и призвана добавить немного свободы в наши тесты, так как она (здесь я не буду привдить её реализацию) предоставляет возможность переопределить следующие методы:

> succeeded

> failed

> starting

> finished

По названиям методов можно догадаться, в какой момент они выполнятся, а именно: начало или конец теста, успешное или не успешное его завершение.

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

#### Verifier
Класс org.junit.rules.Verifier также как и ExternalResource является базовым классом, в котором предполагается самому реализвать лишь один метод verify():

{% highlight java %}

@Override
public void evaluate() throws Throwable {
    base.evaluate();
    verify();
}

{% endhighlight %}

##### ErrorCollector 
В качестве примера использования Verifier рассмотрим рулу org.junit.rules.ErrorCollector, которая позволяет 
"продолжить выполнение теста после первой ошибки". Использование этой рулы позволит, например, собрать все ошибки произошедшие в тесте в одном отчёте. Хотя при правильном формировнии тесткейсов (один тест - одна проверка) необходимости в токой руле нет. 

{% highlight java %}

@Rule
public ErrorCollector collector= new ErrorCollector();

@Test
public void example() {
    collector.checkThat("Должны совпадать", "1", is("3"));
    collector.checkThat("Должны совпадать", "1", is("1"));
    collector.checkThat("Должны совпадать", "1", is("2"));
}

{% endhighlight %}

Метод checkThat( ... ) является обёрткой для стандартной проверки assertThat( ... ), но в отличии от последнего, если проверка не прошла, не прерывает выполнение теста. Результатом такого кода будет отчёт, который содержит в себе ошибки первой и третьей проверки.

#### ExpectedException
Проверка кода на предмет правильной работы в исключительных ситуациях является одной из важных задач в тестировании. В JUnit есть возможность проверить, что в процессе выполнения бросается нужное исключение:

{% highlight java %}

@Test(expected=NullPointerException.class)
public void throwsNullPointerExceptionWithMessage() { ... }

{% endhighlight %}

Рула org.junit.rules.ExpectedException расширяет этот функционал и позволяет проверить не только класс бросаемого исключения, но и, например, его сообщение:

{% highlight java %}

@Rule
public ExpectedException thrown = ExpectedException.none();

@Test
public void throwsNullPointerExceptionWithMessage() {
    thrown.expect(NullPointerException.class);
    thrown.expectMessage("happened?");
    thrown.expectMessage(startsWith("What"));
    throw new NullPointerException("What happened?");
}

{% endhighlight %}

#### RuleChain
Если в тесте есть несколько рул, то вероятнее всего (на самом деле нет) они будут выполняться в том порядке, в котором встречаются в коде. В действительности, порядок, в котором они будут вполняться, зависит от реализации JVM.
В случае, когда нужно вызвать несколько рул в строго определённом порядке на помощь приходит рула org.junit.rules.RuleChain, которая позволяет задать порядок выполнения:

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

В результате получим порядок:
> starting outer rule

> starting middle rule

> starting inner rule

> finished inner rule

> finished middle rule

> finished outer rule


Подробней про Рулы можно узнать из репозитория проекта на гитхабе https://github.com/junit-team/junit
