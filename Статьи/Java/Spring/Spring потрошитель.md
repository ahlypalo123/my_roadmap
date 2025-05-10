
![](https://habrastorage.org/r/w1560/webt/xr/ow/9s/xrow9stjbnims1kax70cnjcgnbg.jpeg)

Технические доклады могут быстро устаревать и становиться невостребованными. Но со «Spring-потрошителем» Евгения Борисова получилось совсем иначе: мы провели мероприятие и опубликовали запись ещё 11 лет назад, а её просмотры по-прежнему растут, и уже перевалили за за 500 000. Перед собеседованиями этот доклад порой штудируют обе стороны: соискатели — чтобы подтянуть матчасть, работодатели — чтобы задать заковыристые вопросы.

В общем, получился главный доклад русскоязычного Java-сообщества. И теперь мы решили, что ему будет полезна ещё и текстовая версия на Хабре. Да, что-то в материале устарело (там речь заходит ещё про Java 7), так что делайте поправку на возраст. Но раз этот материал продолжают смотреть в видеоформате, то и возможность делать Ctrl+F кому-то наверняка пригодится.

А если кому-то хочется более свежих Java-докладов — мы тем временем вовсю готовим конференцию [**JPoint 2025**](https://jpoint.com/?utm_source=habr&utm_medium=893638) (пройдёт уже 3-4 апреля).

Далее повествование ведется от лица спикера.

* * *

Привет, ребята! Я занимаюсь Java с 2001 года, последние пару лет я ушел в «свободные художники». Страдаю от аллергии на весну, но при этом люблю Spring. Вот такой парадокс.

Сегодня мы поговорим с вами:

* про составляющие Spring и его жизненный цикл,
    
* про четыре вида контекста, которые существуют, и постараемся написать пятый,
    
* про то, как сделать различные нестандартные сложные вещи,
    
* про то, как Spring бьет по нашей производительности.
    

Можно было бы озаглавить этот доклад «Spring в картинках» — на этой иллюстрации можно изучить все, что существует:

![](https://habrastorage.org/r/w1560/webt/ju/ob/yi/juobyinvgel9ea-q_odwf1y5ng8.png)

Я постарался тут визуализировать все внутренние кишки Spring. Их мы и будем обсуждать. Некоторые вещи мы напишем по ходу дела. Давайте разбираться.

### С чего все начинается

26 ноября 2003 года появился **XmlBeanDefinitionReader** — внутренний компонент Spring, позволивший настраивать контекст при помощи XML, где мы прописываем бины. Он сканирует XML и всё, что мы там пишем, и переводит в BeanDefinition (объекты, которые хранят в себе информацию про бины).

Давайте посмотрим, как изначально декларировался бин в Spring. У нас будет интерфейс Quoter — это цитатник с методом `sayQuote()`

```java
package quoters;

public interface Quoter {
    void sayQuote();
}

```

И напишем имплементацию «TerminatorQuoter»:

```java
public class TerminatorQuoter implements Quoter {

    private String message;

    public void setMessage(String message) {
        this.message = message;
    }

    @Override
    public void sayQuote() {
        System.out.println("message = " + message);
    }
}

```

У него будет проперти — строка message. И сделаем для него сеттер: для настройки через XML это обязательно, потому что если сделать просто поле без сеттера, то с точки зрения XML-ного Spring это просто не проперти. Он попытается всё равно вызвать сеттер через рефлекшен, но его не будет, и всё упадёт. В методе `sayQuote()` мы распечатываем эту цитату.

Теперь мы пойдем в наш XML-файл. В нём открывается и закрывается тег &lt;beans&gt;, а внутри я прописываю все свои бины.

```xml
<?xml version="1.0" encoding="UTF-8?>
<beans
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"    xsi:schemaLocation="http://www.springframework.org/schema/beans">
	<bean class = "quoters.TerminatorQuoter" id = "terminatorQuoter" >
		<property name="message" value="I'll be back"/>
	</bean>
</beans>
```

Вот я прописал бин, который будет из класса `terminatorQuoter`, дадим ему на всякий случай id и пропишем property. Поскольку у нас есть сеттер, IDE в подсказке уже предвидит, что можно через указание `message` дать какое-то значение. Значение у нас будет "I'll be back".

На этом декларация бина окончена.

Сейчас мы для простоты создадим класс Main (вообще тестировать надо иначе, конечно). А в нём создадим новый `ClassPathXmlApplicationContext`. Имплементация этого контекста как раз анализируется и сканируется при помощи XmlBeanDefinitionReader, про который мы поговорили. Я должен передать в параметрах название файла `context.xml`.

Дальше можно будет вытащить из этого контекста бин. Чисто в теории, бины можно вытаскивать как по классу, так и по интерфейсу. **Сейчас я вытащу по классу, чтобы потом объяснить, почему это неправильно**.

```java
public class Main {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context =
            new ClassPathXmlApplicationContext("context.xml");
        context.getBean(TerminatorQuoter.class).sayQuote();
    }
}

```

Сразу запустим метод `sayQuote()`, проверим, что все работает, и пойдем дальше. При запуске в консоли у нас выводится фраза «I'll be back», всё замечательно.

### Как всё работает

Продолжаем разговор. Как это всё работало в 2003 году? (Потом много вещей накрутилось, до них ещё дойдём).

Давайте посмотрим на эту схему:

![](https://habrastorage.org/r/w1560/webt/qn/ea/cb/qneacblmgjsbf2kdnu_q9iajtqk.png)

Мы пишем наши классы. Центральный игрок Spring — BeanFactory, который отвечает за создание и хранение всех синглтон-объектов. Его я представляю в виде пчелки. А наш древний старый XML выглядит как древний свиток, мы там прописываем какой-то бин из какого-то класса.

Когда мы поднимаем контекст, первое, что происходит — приходит уже рассмотренный нами BeanDefinitionReader, считывает из XML все декларации бинов и кладет их в map. В этой map у нас ID бина соответствует его декларации, в которую входит:

* из какого класса его надо создавать,
    
* есть ли у него init-метод и как он называется,
    
* какие у него проперти,
    
* и все другие подробности бина, прописанные в XML.
    

После того, как BeanDefinitions созданы, BeanFactory начинает по ним работать: создает из наших классов объекты и складывает все бины в контейнер. Тут важно знать, что если бин является синглтоном, то по умолчанию он создается изначально, как только поднимается контекст. А все прототайпы создаются в тот момент, когда они нужны. Кто-то запросил прототайп — тогда Spring его создал, настроил, отдал и забыл про него.

Это важно знать, потому что если вы, например, прописываете destroy-метод для бина, то для синглтона этот метод работать будет, а для прототайпа — нет. Когда контекст закрывается, Spring проходит по всем бинам, которые там хранятся (а это только синглтоны), находит их destroy-методы, если они прописаны, и запускает. А прототайпы Spring нигде не хранит, и поэтому destroy-метод для них работать не будет.

В конце мы получаем полностью настроенные объекты.

### BeanPostProcessor

Следующая вещь в кишках Spring — это **BeanPostProcessor**. Он позволяет настраивать наши бины до того, как они попали в контейнер. Есть такой паттерн проектирования — chain of responsibility. Он здесь как раз задействован.

Посмотрим, что я могу сделать. Я сейчас хочу кастомизировать Spring, обучить его собственным аннотациям и написать BeanPostProcessor, который будет что-то подкручивать в бине при его создании.

Представьте себе, что я пишу приложение, в котором есть очень много генераций случайных чисел. И я хочу, чтобы эти случайные значения записывались в поля. Я вижу, что в моей команде каждый делают по-своему: кто-то использует `Math.random()`, кто-то `java.util.Random.nextInt()`, кто-то библиотеку скачал.

Я говорю: «Так, это никуда не годится. Давайте делать это декларативно, мы сейчас придумаем аннотацию [@InjectRandomInt](/users/InjectRandomInt) и будем ставить её над теми полями, куда нам нужно инжектнуть случайное число. Затем мы научим Spring относиться к этой аннотации, и в момент создания бина настраивать его с учетом этой аннотации».

Давайте это реализуем. Начнем с аннотации. Пойдем в наш класс TerminatorQuoter, добавим там поле `int` под названием `repeat`: сколько раз надо повторять цитату. И я хочу, чтобы это поле задавалось аннотацией InjectRandomInt, которую мы прямо сейчас придумаем. У неё будет два параметра, min и max (для примера дадим им значения 2 и 7). А в методе `sayQuote()` добавим цикл с соответствующим числом итераций:

```java
public class TerminatorQuoter implements Quoter {
    
  @InjectRandomInt(min = 2, max = 7)
  private int repeat;
  private String message;
  
  public void setMessage(String message) {
    this.message = message;
  }
  
  @Override
  public void sayQuote() {
    for (int i = 0; i < repeat; i++) {
      System.out.println("message = " + message);
    }
  }
}
```

Теперь нужно создать саму аннотацию, IDEA в контекстном меню уже предлагает это сделать. При создании аннотаций очень важно не забыть поменять retention policy на RUNTIME:

```java
import …

@Retention(RetentionPolicy.RUNTIME)
public @interface InjectRandomInt {
}
```
    

По умолчанию там стоит CLASS. Всего есть три варианта:

* **SOURCE** говорит о том, что эта аннотация видна исключительно в исходниках, а когда вы компилируете, в байткоде уже ничего не будет. Например, overwrite — это аннотация такого плана. Она нужна только в процессе разработки кода.
    
* **CLASS** говорит, что аннотация в байткод попасть должна, но при этом все равно через reflection в рантайме вы ее считать не сможете, ее там не будет. Это нужно для вещей вроде AST-трансформаций и инструментирования байткода.
    
* **RUNTIME** стоит у большинства аннотаций, которые видны в рантайме, их можно считать через reflection. Поэтому первое, что я сделал — в настройках поменял дефолтный вариант на RUNTIME, поскольку других не пишу обычно.
    

Мы уже сказали, что у данной аннотации есть два параметра, их мы тоже отмечаем.

```java
import …@Retention(RetentionPolicy.RUNTIME)public @interface InjectRandomInt { 	int min();	
    int max();
}
```


Теперь всё компилируется. Но когда я запущу всё это дело, в консоли будет пусто, потому что никто не знает аннотацию «InjectRandomInt». Соответственно, мой параметр repeat — это ноль, и цитата терминатора напечатается ноль раз. Достаточно логично.

Теперь мы будем обучать. Мы сейчас создадим класс, который имплементирует интерфейс, который будет отвечать за обработку всех бинов, классы которых имеют эту аннотацию хотя бы в каком-то поле. Называться он будет «InjectRandomIntAnnotationBeanPostProcessor». Ну, вы в курсе, что при придумывании внутренних компонентов Spring класс с названием меньше 20 букв — это просто несерьезно! Я даже не шучу, для обработки [@Autowired](/users/Autowired) в Spring вполне есть AutowiredAnnotationBeanPostProcessor, так что я тут просто соблюдаю конвенцию.

Этот наш класс имплементирует интерфейс BeanPostProcessor. У этого интерфейса нужно имплементировать два метода:

* postProcessBeforeInitialization (Object bean, String beanName) Вызывается до init-метода
    
* postProcessAfterInitialization (Object bean, String beanName) Вызывается после init-метода
    

О том, почему их два и нельзя было ограничиться одним, поговорим чуть позже.

Оба метода имеют одинаковую сигнатуру: в метод придет бин и его имя. Соответственно, этот метод вызовется для каждого бина. В этом методе я могу вернуть какой-то объект. Теоретически, могу вернуть вообще не тот, который мне дал BeanFactory. Но пока что мы не станем извращаться, будем возвращать в обоих методах полученный объект bean.

Однако, допустим, я хочу в методе postProcessBeforeInitialization взять бин, вытащить его класс с помощью `bean.getClass()`, вытащить все его поля с помощью `getDeclaredFields()` и пройтись по ним. У каждого поля мы постараемся вытащить аннотацию с названием «InjectRandomInt.class». Проверим: если эта аннотация не равна null, значит, она над соответствующим полем стояла, и тогда мне из неё надо вытащить min и max. Затем нам нужно сгенерировать случайное число между min и max.

```java
public class InjectRandomIntAnnotationBeanPostProcessor
    implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
        throws BeansException {
        Field[] fields = bean.getClass().getDeclaredFields();
        for (Field field : fields) {
            InjectRandomInt annotation = field.getAnnotation(
                InjectRandomInt.class
            );
            if (annotation != null) {
                int min = annotation.min();
                int max = annotation.max();
                Random random = new Random();
                int i = min + random.nextInt(max - min);
            }
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
        throws BeansException {
        return bean;
    }
}

```

Здесь i будет случайным числом, которое мне теперь нужно поместить в поле. Для этого требуются две вещи. Во-первых, если код написан правильно, то поле будет private, так что надо указать `field.setAccessible(true)`. Второе — мы могли бы просто написать `field.set(i)`, **но мы так делать не будем**. Потому что в этом случае нам придется обрабатывать исключения. Поскольку мы имплементируем чужой интерфейс, если он не кидает исключения, то и мы не можем сделать `throws`. А постоянные `try` и `catch` плохо сказываются на нервной системе.

Поэтому мы воспользуемся прекрасной библиотекой, которая есть у Spring — ReflectionUtils, которая умеет делать обычные reflections, которые вы знаете, только без try и catch (на самом деле, библиотека просто оборачивает их и прячет в RuntimeException). В этом случае метод set.field принимает на один параметр больше: я должен указать, для какого поля я буду давать значение (field), затем для какого объекта его нужно будет засунуть (для моего bean), и наконец, само значение (это i). Вот и всё, никаких try и catch.

```java
public class InjectRandomIntAnnotationBeanPostProcessor
    implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
        throws BeansException {
        Field[] fields = bean.getClass().getDeclaredFields();
        for (Field field : fields) {
            InjectRandomInt annotation = field.getAnnotation(
                InjectRandomInt.class
            );
            if (annotation != null) {
                int min = annotation.min();
                int max = annotation.max();
                Random random = new Random();
                int i = min + random.nextInt(max - min);
                field.setAccessible(true);
                ReflectionUtils.setField(field, bean, result);
            }
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
        throws BeansException {
        return bean;
    }
}

```

Следующий вопрос. Вот я написал этот замечательный класс. Что мне нужно сделать, чтобы Spring про него узнал, и этот BeanPostProcessor являлся частью системы, которая создает и настраивает мои бины?

Просто прописать его в контекст! У Spring в этом плане всё очень удобно: практически любую вещь, которую вы хотите добавить в Spring, вы прописываете в контекст. Причем у нас есть разные варианты: можно в XML, если мы работаем с ним, можно в Java config, можно через аннотации.

Мы сейчас работаем с XML. Вот я указываю бин из класса «InjectRandomInt».

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="quoters.InjectRandomIntAnnotationBeanPostProcessor"/>
    <bean class="quoters.TerminatorQuoter" id="terminatorQuoter">
        <property name="message" value="I'll be back"/>
    </bean>
</beans>
```

ID я ему давать не буду: это инфраструктурный бин, а значит, инжектить мы его никуда не будем. Конечно, Spring придумает ему какой-то ID, но мне это даже неинтересно.

Теперь мы запускаем то же самое приложение. Каждый раз при создании бина ему в поле `repeat` будет инжектиться рандомное значение в заданных пределах.

Init-методы и двухфазный конструктор  
Идем дальше. Как я говорил, у нас есть два метода `postProcessBeforeInitialization()` и `postProcessAfterInitialization()`: в одном я написал логику, в другом я ничего пока не сделал. А между этими двумя методами вызывается init-метод, который можно прописать несколькими способами в зависимости от вашей ситуации.

![](https://habrastorage.org/r/w1560/webt/im/j7/nu/imj7nunzwzjfdkecolxspfmzarm.png)

При работе с XML можно прописать атрибут init-method в тэге bean, при работе с аннотациями можно использовать [@PostConstruct](/users/PostConstruct). А если вы работаете со Spring 2, можно использовать метод afterPropertiesSet, но так никто уже давно не делает.

Возникает вопрос. В init-методе мы пишем некоторую логику, которая инициализирует bean, но для этого всегда существовал конструктор. Чем он стал плох?

Сейчас я расскажу вам про двухфазный конструктор. Но прежде, чем мы приступим, давайте сначала посмотрим, что произойдет, если я попытаюсь в конструкторе пользоваться чем-то, что мне настраивает Spring.

Вернусь к своему классу TerminatorQuoter. У меня там уже есть конструктор, и я хочу напечатать в нем значение параметра repeat. Что там будет напечатано?

![](https://habrastorage.org/r/w1560/webt/qb/lc/yp/qblcypjnjjdka3aidb_egl3moew.png)

Ноль. Причем в логе у меня фраза повторяется 4 раза, а значение всё ещё стоит ноль. Почему?

Spring не делает никакой магии. Сначала объект создается Java, Spring этот процесс инициализирует: просканировался XML, создались BeanDefinition, Spring понял, что нужно создать синглтон, который называется TerminatorQuoter. При помощи reflection Spring запустил его конструктор, конструктор отработал, объект создался. И когда объект уже создан, Spring его может настраивать.

Соответственно, если мы в конструкторе пытаемся обратиться к каким-то вещам, которые должен настроить Spring, **этих вещей на этом этапе еще нет**, и мы получим в лучшем случае NullPointerException, а в худшем нули. NullPointerException лучше, потому что _лучше знать, что есть проблема, чем с каким-то нулем жить_.

Поэтому что мы делаем? Вместо того, чтобы пользоваться конструктором, мы можем написать метод `init()` и в него поставить ту логику, которую я привык раньше ставить в конструктор. Конструктор мы оставим и разделим на фазу 1 и фазу 2.

```java
private int repeat;
private String message;

public void init() {
  System.out.println("Phase 2");
  System.out.println(repeat);
}

public TerminatorQuoter() {
  System.out.println("Phase 1"); < ... > 
```


Как я скажу, что это init-метод? Как я сказал, есть несколько вариантов. Можно поставить аннотацию PostConstruct, но на этом этапе она работать не будет. По умолчанию XML ничего не знает ни про какие аннотации, но про них знают BeanPostProcessor. Помните, мы написали свой BeanPostProcessor, и сразу аннотация «InjectRandomInt» стала обрабатываться? Точно такая же ситуация и с аннотацией PostConstruct: ее должен обрабатывать какой-то BeanPostProcessor.

Поэтому, если сейчас я запущу этот код, то сообщение "Phase 1" будет выведено в консоль, а "Phase 2" — не будет.

А вот если добавлю в XML бин из класса «CommonAnnotationBeanPostProcessor»,то фаза 2 подключится, а переменная repeat уже будет проинициализирована.

   ```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="org.springframework.context.annotation.CommonAnnotationBeanPostProcessor"/>
    <bean class="quoters.InjectRandomIntAnnotationBeanPostProcessor"/>
    <bean class="quoters.TerminatorQuoter" id="terminatorQuoter">
        <property name="message" value="I'll be back"/>
    </bean>
</beans>
```

![](https://habrastorage.org/r/w1560/webt/wq/xk/m1/wqxkm1yirsysakpdrcrtwowisko.png)

Аудитория предложила включить Annotation Config. Это менее рабочий вариант: дело в том, что вы не запомните такое количество имен наизусть. Кроме ваших собственных BeanPostProcessor, которые вы добавите для кастомной логики, есть еще штук пять-шесть уже существующих, которые относятся к тем аннотациям, которые вы уже знаете. Ради каждой аннотации прописывать в контекст BeanPostProcessor — это с ума сойти можно, чтобы их все запомнить.

Поэтому придумали namespace, и обычно вместо того, чтобы его писать, человек пишет [context:annotation-config/](https://context:annotation-config/) и отпускает ситуацию. На деле этот namespace прячет кусок XML, который добавляет в контекст все BeanPostProcessor — не только CommonAnnotation, но и еще штук пять.

Еще можно сделать так: есть namespace [context:component-scan/](https://context:component-scan/), который принимает какой-то пакет. При его использовании этот пакет просканируется, и в контекст еще добавятся все BeanPostProcessor.

Давайте подведем итоги и посмотрим, как работает всё сейчас, когда все BeanPostProcessor есть. Все начинается с того, что  
BeanDefinition Document Reader прочитал наш XML,  
вытащил BeanDefinition,  
BeanFactory вытащил из этих BeanDefinition определение BeanPostProcessor, создал их и положил в сторонку — он знает, что с его точки зрения это необычные бины, с их помощью он потом будет настраивать все остальные бины.

Поэтому все это выглядит так: бин прошел первые этапы настройки с помощью BeanPostProcessor, после этого у данного бина вызвался init-метод (отработал PostConstruct), и потом идет еще один проход. В конце у нас появляются полностью настроенные при помощи BeanPostProcessor объекты.

![](https://habrastorage.org/r/w1560/webt/yw/wl/_c/ywwl_cuottkcfmtm_zosimw9zec.png)

### Dynamic Proxy

Возникает вопрос. Зачем нужны два прохода по BeanPostProcessor? Неужели было мало одного?

Сейчас я напишу довольно сложный BeanPostProcessor, который будет делать профайлинг, который мы к тому же сможем отключать через JMX Console.

Идея такая: я хочу, чтобы все классы, над которыми стоит аннотация [@Profiling](/users/Profiling), профилировались, чтобы их методы профилировались. То есть я хочу, чтобы в лог выводилось время работы метода. Как мы будем это технически реализовывать?

Понятно, что у нас будет какой-то BeanPostProcessor, который будет к этой аннотации Profiling относиться: он будет получать бин от BeanFactory, уточнять, а не стоит ли над классом этого бина аннотация Profiling, и если стоит, то ему придется делать очень сложную работу: в каждый метод данного бина дописывать логику, связанную с Profiling. Сама логика несложная: замерить время до, запустить метод, замерить время после и вывести разницу на экран. Но как можно добавить логику в уже существующий объект?

Если бы вы работали на Groovy, вы бы просто взяли существующий объект и добавили туда логику. В Java нужно будет на лету сгенирировать новый класс и его объектом заменить исходный так, чтобы никто не заметил подмены. Какой класс нам подойдет?

Представьте, что я BeanPostProcessor и мне дали нашего TerminatorQuoter. Я на него смотрю и говорю: «Ему нужно дать логику, связанную с профилированием» (или с чем-то другим). Окей, мы сейчас создадим новый класс, в котором мы будем делегировать в уже существующие методы и добавлять туда эту логику. Из этого класса я потом создам объект, сделаю некую декорацию, прокси, и верну его обратно в BeanFactory, который не должен заметить, что я подменил объект. Чтобы подмену никто не заметил, новый класс, который сгенерировался на лету, должен либо наследовать от оригинального класса и переопределять его методы, добавляя нужную логику, либо имплементировать те же самые интерфейсы.

То есть существуют два совершенно два разных подхода. Первый подход называется **dynamic proxy** — это имплементировать те же самые интерфейсы. Второй подход называется **CGLib**. В принципе, CGLib считается хуже. Отчасти он уступает в производительности, но главная проблема а в различных ограничениях: вы не от любого класса можете наследовать, есть final классы и методы.

Поэтому Spring всегда предпочитает идти через интерфейсы. Точно так же имплементированы все аспекты: Spring AOP работает именно через прокси, и если спринговому аспекту нужно сделать прокси на какой-то объект, он сначала смотрит, есть ли у него интерфейсы. Если есть, он идет через dynamic proxy, если нет, он идет через CGLib.

Если у нас появляются такие BeanPostProcessor, которые могут взять и заменить оригинальный класс, то могу ли я знать, что произойдет здесь?

```java
public Object postProcessBeforeInitialization(Object bean, String beanName)
    throws BeansException {
    Field[] fields = bean.getClass().getDeclaredFields();
    for (Field field : Fields) {
        InjectRandomInt annotation = field.getAnnotation(InjectRandomInt.class);
        if (annotation != null) {
            int min = annotation.min();
            int max = annotation.max();
            Random random = new Random();
            int i = min + random.nextInt(max - min);
            field.setAccessible(true);
            ReflectionUtils.setField(field, bean, result);
        }
    }
    return bean;
}

@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {}

```


Этот BeanPostProcessor рассчитывает на то, что `get.Class()` вернет оригинальный класс, в котором есть оригинальная метадата и в котором все поля аннотированы теми же самыми аннотациями. Потому что в классе, который сгенерируется на лету, не будет метадаты и аннотаций.

Что получается? По конвенциям Spring, те BeanPostProcessor, которые что-то в классе меняют, должны это делать не на этапе postProcessBeforeInitialization, а на этапе postProcessAfterInitialization. PostConstruct всегда работает на оригинальный метод до того, как все прокси на него накрутились.

Мы сделаем класс `ProfilingHandlerBeanPostProcessor`, который имплементирует `BeanPostProcessor`. У него есть два метода. И еще мы создадим для себя небольшую Map со String против Class — в ней у меня будет лежать имя бина, которое, несмотря ни на что, никогда не меняется. Когда я из бина буду вытаскивать `get.Class()`, я никогда не знаю, получу ли я оригинальный класс или прокси, но имя бина всегда сохраняется. Один из способов работы с этом — на этапе `postProcessBeforeInitialization` запоминать оригинальные классы бинов, с которыми что-то надо сделать, а на этапе `postProcessAfterInitialization` это что-то делать.

    

```java
public class ProfilingHandlerBeanPostProcessor implements BeanPostProcessor {
    
  private Map <String, Class> map = new HashMap<>();
  
  @Override 
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException
    return null;
  }

  @Override public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException
    return null;
  }
}
```

На этапе postProcessBeforeInitialization мы ничего не будем портить: получили бин, его же и вернули. Сделаем только проверку: если у класса есть аннотация [@Profiling](/users/Profiling), то его имя и сам класс добавим в map.

```java
public class ProfilingHandlerBeanPostProcessor implements BeanPostProcessor {

    private Map<String, Class> map = new HashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
        throws BeansException {
        Class<?> beanClass = bean.getClass();

        if (beanClass.isAnnotationPresent(Profiling.class)) {
            map.put(beanName, beanClass);
        }
        return bean;
    }
}

```

    

На этапе postProcessAfterInitialization я буду проверять, есть ли в мэпе имя этого бина. Если beanClass у нас не null, значит, я его запомнил на предыдущем этапе. А если я его запомнил, значит, над ним стояла аннотация Profiling, и я буду делать return не оригинальному объекту, а объекту, который сгенерирую при помощи dynamic proxy.

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName)
    throws BeansException {
    Class beanClass = map.get(beanName);
    if (beanClass != null) {
        return Proxy.newProxyInstance();
    }
    return bean;
}

```

    

Вряд ли вы хотите вручную заниматься такой низкоуровневой работой, как генерация классов на лету. Поэтому в Java возможности для более удобной работы с этим добавили ещё в 1999-м. Метод newProxyInstance создает объект из нового класса, который он же сам на лету и сгенерирует. Этот метод принимает три вещи:

* classLoader, при помощи которого класс, который сгенерируется на лету, загрузится в heap в Java 8 (в perm в Java 7);
    
* список интерфейсов, который должен имплементировать тот класс, который сгенерируется на лету;
    
* InvocationHandler — некий объект, который будет инкапсулировать логику, которая попадет во все методы класса, который сгенерируется на лету.
    

Откуда я возьму ClassLoader? Мы возьмем его от бина, потому что любой класс знает, какой classLoader его загрузил, и поскольку мне это не принципиально, пусть оригинальный и загружает. Список интерфейсов я тоже беру из бина, потому что я сейчас создаю новый класс, который должен имплементировать те же интерфейсы, что и оригинальный класс бина. И InvocationHandler мы пропишем вот так:

```java
return Proxy.newProxyInstance(beanClass.getClassLoader(), beanClass.getInterfaces(), new InvocationHandler() {
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {<...>
```

Чтобы стало совсем интересно, мы напишем класс, который будет называться ProfilingController. У него будет булевый флаг «включен/выключен». Я сделаю так, чтобы этот флаг можно было включать и выключать на лету в JMX Console с помощью MBean.

Не всегда же хочется видеть результаты профилирования в логе. Это и на производительности сказывается, и лог засоряет. Но иногда прибегают «злые дяди» и говорят «продакшен тормозит». Вот тогда можно сказать «минуточку», включить флажок и посмотреть: «ага, проблема в базе данных, тормозят все операции, связанные с ней».

Поэтому сделаем класс `ProfilingController`:

```java
public class ProfilingController implements ProfilingControllerMBean {

    private boolean enabled;

    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }
}

```

Чтобы включать и выключать этот флажок, используем старую конвенцию MBean, позволяющую менять через JMX Console все зарегистрированные объекты. Для этого мы создаём интерфейс ProfilingControllerMBean и указываем в нём те методы, про которые хотим, чтобы они были доступны через JMX Console.

```java
public interface ProfilingControllerMBean {
    void setEnabled(boolean enabled);
}

```

Возвращаемся в наш BeanPostProcessor, он будет в себе держать наш новый класс ProfilingController. По умолчанию он выключен.

```java
public class ProfilingHandlerBeanPostProcessor implements BeanPostProcessor {

    private Map<String, Class> map = new HashMap<>();
    private ProfilingContoller controller = new ProfilingController();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
        throws BeansException {
        Class<?> beanClass = bean.getClass();
        if (beanClass.isAnnotationPresent(Profiling.class)) {
            map.put(beanName, beanClass);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
        throws BeansException {
        Class beanClass = map.get(beanName);
        if (beanClass != null) {
            return Proxy.newProxyInstance(
                beanClass.getClassLoader(),
                beanClass.getInterfaces(),
                new InvocationHandler() {}
            );
        }
        return bean;
    }
}

```
    

Напоминаю: я сейчас здесь в Proxy.newProxyInstance() напишу логику, которая будет в каждом методе класса, который сгенерируется на лету и имплементирует интерфейсы оригинального класса.

Мы выведем в консоль сообщения о начале и конце профилирования. А между ними вызовем оригинальный метод, передадим оригинальный бин и аргументы оригинального метода. И затем вернём то, что возвращает этот метод.

```java
public Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable {
    System.out.println("ПРОФИЛИРУЮ");
    Object retVal = method.invoke(bean, args);
    System.out.println("ВСЁ");
    return retVal;
}

```

А ещё замерим время. Используем System.nanoTime() до и после вызова метода, а затем выведем разницу:

```java
public Object postProcessAfterInitialization(final Object bean, String beanName)
    throws BeansException {
  Class beanClass = map.get(beanName);
  if (beanClass != null) {
    return Proxy.newProxyInstance(beanClass.getClassLoader(), beanClass.getInterfaces(), new InvocationHandler() {
      @Override
      public Object invoke(Object proxy, Method method, Object[] args)
          throws Throwable {
        System.out.println("ПРОФИЛИРУЮ");
        long before = System.nanoTime();
        Object retVal = method.invoke(bean, args);
        long after = System.nanoTime();
        System.out.println("after-before");
        System.out.println("ВСЁ");
        return retVal;
      }     
	}
  }
```
    

Вот это всё должно работать в случае, если контроллер был enabled. Поэтому я всё это заверну в if, и в нем поставим условие — если включен контроллер.

```java
public Object invoke(Object proxy, Method method, Object[] args)
    throws Throwable {
  if (controller.isEnabled()) {
    System.out.println("ПРОФИЛИРУЮ");
    long before = System.nanoTime();
    Object retVal = method.invoke(bean, args);
    long after = System.nanoTime();
    System.out.println("after-before");
    System.out.println("ВСЁ");
    return retVal;
  } else {
    return method.invoke(bean.args);
  }
}
```
    

То есть при включенном контроллере метод должен работать описанным нами образом, а если он выключен — просто сделегировать в оригинальный метод, в этом случае мы не вмешиваемся в происходящее..

Мы еще не закончили с MBean. Я сейчас создал этот контроллер в формате MBean, но еще не зарегистрировал в MBeanServer. Поэтому это тоже надо сделать, и мы это сделаем в конструкторе. Напишем конструктор нашему уважаемому ProfilingHandlerBeanPostProcessor. В конструкторе мы сделаем следующее: вызываем  
`ManagementFactory.getPlatformMBeanServer()`.

ManagementFactory — это стандартный Java-класс, у которого есть getPlatformMBeanServer. То есть я получаю инстанс этого MBeanServer, в котором можно регистрировать бины. Напоминаю, конкретно со Spring это всё не связано.

Как регистрируется бин? Таким образом: я передаю в метод registerMBean() свой контроллер, и дальше надо дать ему какое-то имя, чтобы потом через JMX Console можно было комфортно найти. Имя типа ObjectName по конвенции состоит из двух вещей. Сначала домен, под какой папочкой в JMX Console он будет находиться, назовем это «profiling». И следующее — само имя. Будет называться «controller».

```java
public ProfilingHandlerBeanPostProcessor() throws Exception {
  MBeanServer platformMBeanServer = ManagementFactory.getPlatformMBeanServer();
  platformMBeanServer.registerMBean(
      controller, new ObjectName("profiling", "name", "controller"));
}
```
    

Имейте в виду, у нас тут может быть двести тысяч exceptions. IDE может предложить MalformedObjectNameException, я упростил до просто Exception.

Теперь нам осталось зарегистрировать наш контроллер в контексте.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <context:annotation-config/>
    <bean class="quoters.ProfilingHandlerBeanPostProcessor"/>
    <bean class="quoters.InjectRandomIntAnnotationBeanPostProcessor"/>
    <bean class="quoters.TerminatorQuoter" id="terminatorQuoter">
        <property name="message" value="I'll be back"/>
    </bean>
</beans>
```
    

Ещё надо кое-что поменять в файле main, чтобы было видно, как это будет работать. Давайте завернём вызов `sayQuote()` в `while (true)` и добавим к нему `Thread.sleep(100)`.

```java
public class Main {
  public static void main(String[] args) throws InterruptedException {
    ClassPathXmlApplicationContext context =
        new ClassPathXmlApplication while (true) {
          Thread.sleep(100);
          context.getBean(TerminatorQuoter.class).sayQuote();
        }
  }
}
```
    

Теперь запускаем всё это дело. Он сейчас нам должен писать i’ll be back — но не пишет, потому что у нас что-то упало.

![](https://habrastorage.org/r/w1560/webt/4d/mx/cn/4dmxcn43lsf3qtpgywxa3peysxu.png)

Помните, я вам специально говорил, что буду делать lookup по классу, чтобы объяснить вам, что это неправильно? Вот это и выстрелило. Без этого забыл бы вам рассказать.

Давайте попытаемся поставить breakpoint и посмотреть, что не так. В контексте есть несколько полезных методов, которые при дебаге помогают смотреть, что происходит. Например, `context.getBeanDefinitionNames()` покажет нам имена всех бинов, которые есть.

![](https://habrastorage.org/r/w1560/webt/q4/gs/rs/q4gsrs2suuwr3sv0deo8fubcjic.png)

Смотрите, как их много. Их так много, потому что мы поставили `context:annotation-config`, и из-за этого все вот эти бины тут появились: AutowiredAnnotationProcessor, CommonAnnotationProcessor. Среди них и мой terminatorQuoter, имя его не поменялось.

Но если я на него вызову `context.getBean(Quoter.class).getClass()`, то увижу, что название класса совсем другое: `com.sun.proxy.$Proxy7`. Видите, какое старое? Помните такую компанию Sun?

Поэтому мне и в коде main надо делать lookup не по классу (потому что я никогда не знаю, что с ним случится), а по интерфейсу. Поэтому заменяем TerminatorQuoter.class на Quoter.class.

```java
public class Main {
  public static void main(String[] args) {
    new ClassPathXmlApplicationContext context =
        new ClassPathXmlApplicationContext("context.xml");
    context.getBean(Quoter.class).sayQuote();
  }
}
```
    

Сейчас у нас выключен флажок профилирования. Поэтому, когда мы запустим этот код, в логе просто будет выводиться фраза «I’ll be back».

Теперь давайте включим профилирование. У меня сейчас используется Java 7. Знаете jvisualvm (Java VisualVM)? Возьмём его, подключимся к текущему Java-процессу. Есть плагин, который называется MBeans, я его добавил, это можно сделать через Tools — Plugins.

В нём у меня видно пункт profiling. Когда я открою этот профайлинг, я увижу controller — это название моего MBean. Зайдем в него, и у него в списке Attributes есть атрибут «enabled». Я напишу true напротив него в колонке value и возвращаюсь обратно в IntelliJ.

Теперь в логе написано «Профилирую…», и даже бегут цифры:

![](https://habrastorage.org/r/w1560/webt/dq/py/to/dqpytopaphe1hycxnsnyb_kwoq4.png)

Допустим, мы попрофилировали, нашли проблему «где тратится много времени», и хотим всё это дело отключить обратно. Мы возвращаемся в jVisualVM, пишем напротив enabled «false», и никто у нас в IDE больше не профилирует. Согласитесь, довольно удобно.

Сейчас я могу аннотацию «Profiling» ставить над абсолютно любым своим бином, и каждый раз, когда я буду включать профилирование, он будет профилироваться. Придумал свою аннотацию, придумал свой BeanPostProcessor, который ее обрабатывает на этапе создания бина. Как вы видите, BeanPostProcessor могут не только бин подкрутить, но они еще и могут поменять логику его класса — если немного знать про dynamic proxy и CGLib.  
ApplicationListener  
Теперь поговорим про трехфазовый конструктор. Но сначала объясню про ApplicationListener, с помощью которого этот конструктор затем сделаю.

![](https://habrastorage.org/r/w1560/webt/y5/ht/pv/y5htpvvfdlu4dhmo8k2gvcab9v0.png)

Listener умеет слушать контекст Spring, все ивенты, которые с ним происходят. А с ним могут произойти разные вещи вроде ContextStarted или ContextStopped. Но самый интересный ивент — это ContextRefreshedEvent. Потому что при событии ContextStarted контекст еще не построился, а только начал строиться. А когда контекст заканчивает свое построение, он всегда делает рефреш. Поэтому в большинстве случаев, если вам нужен Listener, он будет слушать, что контекст «рефрешнулся».

Теперь давайте подумаем, зачем мне это может понадобиться. Приведу жизненный пример: у меня есть сервис, у которого есть метод warmСache(), он должен разогреть свой собственный кэш. Он идет в базу данных, что-то там делает, что-то берет, меняет, возвращает, наполняет свой collection некоторой информацией — и после этого он готов.

Где я должен вызывать этот метод? В конструкторе — однозначно не вариант, потому что на этапе работы конструктора еще вообще ничего не настроено: бин не настроен и в базу данных он явно сходить еще не может. Поэтому пишем PostConstruct, пишем это дело туда — и тут возникает печалька. Потому что работать-то оно работает, но транзакций на этапе работы PostConstruct еще не существует. Они просто не настроены.

Понятно ли, почему? Над нашим методом warmCache() стоит аннотация [@Transactional](/users/Transactional), примерно как с [@Profiling](/users/Profiling). Но в какой момент BeanPostProcessor, который отвечает за эту аннотацию, запихает мне связанную с транзакцией логику? После того, как PostConstruct отработал. Потому что, как помните, есть разные этапы: postProcessBeforeInitialization, потом PostConstruct, а после уже postProcessAfterInitialization. То есть PostConstruct работает до того, как настроились все прокси, включая те прокси, которые отвечают за транзакции.

Что делать? Я хочу иметь третью фазу конструктора. Пойду в свой файл TerminatorQuoter и поставлю над sayQuote аннотацию [@PostConstruct](/users/PostConstruct). Она в этом файле уже есть над init(), получится два PostConstruct в файле, это идеологически неправильно, но работать будет.

```java
@PostConstructpublic
void sayQuote() {
  for (int i = 0; i < repeat; i++) {
    System.out.println("message = " + message);
  }
}
```
    

Включу по умолчанию профилирование: в `ProfilingController` к строке `private boolean enabled` добавлю `=true`.

Я запускаю и вижу, что профилирования в таком виде у меня нет!

Идем обратно в наш main-файл и вызываем оттуда sayQuote():

```java
public class Main {
  public static void main(String[] args) throw InterruptedException {
    ClassPathXmlApplicationContext context =
        new ClassPathXmlApplicationContext("context.xml");
    context.getBean(Quoter.class).sayQuote();
  }
}
```
    

То есть теперь sayQuote() должен сработать дважды, из Main и из PostConstruct. Смотрим на результат и видим, что в первый раз он отработал без профилирования, а во второй раз с профилированием. Почему? Потому что на этапе PostConstruct никаких прокси нету.

Поэтому мы сейчас придумываем еще одну аннотацию в файле TerminatorQuoter, которая будет называться [@PostProxy](/users/PostProxy). И себе для понимания сделаем вывод сообщения о том, что это третья фаза.

```java
@Override
@PostProxypublic
void sayQuote() {
    System.out.println("3 phase");
    for (int i = 0; i < repeat; i++) {
        System.out.println("message = " + message);
    }
}

```
    

Я хочу, чтобы все методы, которые аннотированы [@PostProxy](/users/PostProxy), запускались сами в тот момент, когда абсолютно всё уже настроено и все прокси сгенерировались. Это может делать только ContextListener, потому что только там у меня есть еще один доступ, который позже, чем PostConstruct.

Придумываем аннотацию и пишем listener — здесь будет сложно. Создаем новый класс PostProxyInvokerContextListener, который имплементирует ApplicationListener.

Сейчас случайно написал «inoker» вместо «invoker». Вы в курсе, почему важно писать без ошибок? Перед тем, как что-то написать, ищешь «а вдруг уже это писал». Но если написал с ошибкой и из-за этого не находишь, получаются два дублирующихся метода. Потом ещё в одном из них обнаруживается баг, пытаешься чинить не тот…

Видите, что IDE предлагает использовать дженерик? Есть пять ивентов, но я не хочу слушать все пять — меня интересует только ContextRefreshed. И я не хочу внутри метода, который мне надо переописать, делать instanceof и всё такое. Поэтому мы сразу поставим тут generic и скажем, что слушаем только ContextRefreshedEvent. И в том методе, который я должен имплементировать, у меня уже указан конкретно этот ивент:

```java
public class PostProxyInvokerContextListener
    implements ApplicationListener<ContextRefreshedEvent> {

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {}
}

```
    

Из ивента я могу вытащить ApplicationContext и положить его в сторонку (`ApplicationContext context = event.getApplicationContext()`). Из Context я хочу вытащить имена всех своих бинов (`String[] names = context.getBeanDefinitionNames()`), чтобы по каждому пройтись и проверить, не стояла ли в их классе аннотация [@PostProxy](/users/PostProxy).

Вопрос: могу ли я по имени бина вытаскивать бин и делать у него get.Class? **Однозначно нет**. Потому что на этом этапе там уже будет прокси. Соответственно, когда я сделаю get.Class у того бина, который есть сейчас, это будет $Proxy7 класс, где нет ничего интересного. Поэтому мы делаем по-другому. Мы будем разговаривать с главной фабрикой Spring. Для этого мне нужно её сюда инжектнуть. Добавим `private ConfigurableListableBeanFactory`. Видите, какие солидные названия!

Только фабрика умеет делать `getBeanDefinition()`. Вытаскивать бин бесполезно и неправильно: если вы определяете бин как Lazy, то он не будет создаваться до запроса. А я сейчас буду проходиться по всем бинам, чтобы проверить, есть ли у него метод, который аннотирован [@PostProxy](/users/PostProxy), ведь если есть, то его надо запустить. В таком случае получится, что я создал бин, который не должен был сейчас создаваться. Поэтому неправильно вытаскивать сами бины, а надо только их BeanDefinition и в них искать информацию, которая меня интересует.

```java
public class PostProxyInvokerContextListener
    implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    private ConfigurableListableBeanFactory factory;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        ApplicationContext context = event.getApplicationContext();
        String[] names = context.getBeanDefinitionNames();
        for (String name : names) {
            factory.getBeanDefinition(name);
        }
    }
}

```
    

Кто-то сейчас топнет ногами и скажет, что я сделал ужасную вещь: в какой-то класс инжектнул Spring factory, представляете, какой это coupling. Но то, что я сейчас пишу, — это ApplicationListener, инжектить Spring в Spring вполне нормально. Вот если бы я в TerminatorQuoter инжектнул Context — вот это был бы ужас. Имейте это в виду, не используйте Spring как костыль, постоянно делая lookup из Context.

Идем дальше. Вытащили beanDefinition по имени и будем с ним дальше разговаривать. Из него мы можем вытащить string, который означает оригинальное имя класса бина: `getBeanClassName()`.

```java
for (String name : names) {    
	BeanDefinition beanDefinition = factory.getBeanDefinition(name);   
	String originalClassName = beanDefinition.getBeanClassName();
}
```
    

Я сейчас при помощи `Class.forName()` получу объект класса. От exceptions тут уже никак не уйти. Не буду пустой catch оставлять, грех. А кто здесь не грешил?

У originalClass мы вытаскиваем все методы, проходимся по ним, и если метод аннотирован [@PostProxy](/users/PostProxy), то тогда этот метод надо запустить. Но просто method.invoke здесь не сработает (через CGLib сработал бы, а через dynamic proxy нет). Сейчас мы ищем метод в оригинальном классе, а бин у меня создан из класса прокси. Это два разных класса, поэтому мне нужно вытащить метод у текущего класса. И для этого мне нужно сначала вытащить сам бин.

Поэтому мы сейчас обратимся к моему контексту и скажем «ну-ка дай мне бин по вот этому имени»:

```java
for (String name : names) {    
	BeanDefinition beanDefinition = factory.getBeanDefinition(name);    
	String originalClassName = beanDefinition.getBeanClassName();    
	try {        
		Class<?> originalClass = Class.forName(originalClassName);        
		Method[] methods = originalClass.getMethods();        
		for (Method method : methods) {            
			if (method.isAnnotationPresent(Postproxy.class)) {                
				Object bean = context.getBean(name);            
			}        
		}    
	}
}
```
    

Теперь мы возьмем бин и вытащим из него его реальный нынешний класс с прокси, у него вытащим метод. Напоминаю, у нас есть два класса $Proxy7 и TerminatorQuoter, и они очень похожие, у них совпадающие имена и сигнатуры методов.

Есть метод `getMethod()` — он принимает имя метода и его параметры (и кидает, конечно, какой-нибудь exception). Сохраним как currentMethod то, что он нам вернёт. И затем я могу запустить этот currentMethod, причем даже без ReflectionUtils, потому что try и catch у меня уже стоят.

```java
if (method.isAnnotationPresent(Postproxy.class)) {    
	Object bean = context.getBean(name);    
	Method currentMethod = bean.getClass().getMethod(method.getName(), method.getParameterTypes());    
	currentMethod.invoke(bean);
}
```
    

Запускаем его на этот бин без всяких аргументов.

Идем в контекст, там регистрируем bean из класса PostProxyInvokerContextListener.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <context:annotation-config/>
    <bean class="quoters.PostProxyInvokerContextListener"/>
    <bean class="quoters.ProfilingHandlerBeanPostProcessor"/>
    <bean class="quoters.InjectRandomIntAnnotationBeanPostProcessor"/>
    <bean class="quoters.TerminatorQuoter" id="terminatorQuoter">
        <property name="message" value="I'll be back"/>
    </bean>
</beans>
```



Запускаем — и все хорошо!

![](https://habrastorage.org/r/w1560/webt/zt/wm/05/ztwm05twhhlrbja0owgg6nbk0jw.png)

Сначала идет фаза 1 — обычный конструктор.  
Потом идет фаза 2, PostConstruct, напечаталась 6.  
Потом идет фаза 3, в которой уже идет профилирование с бенчмарком.  
Итоги трехфазового конструктора  
Первый конструктор — Java, второй — PostConstruct, за который отвечает BeanPostProcessor, третий — AfterProxy, за который отвечает ContextListener.

![](https://habrastorage.org/r/w1560/webt/qn/-o/vi/qn-ovis_wbuwjvbcrsf1sdndhym.png)

### Заключение

Это — первая часть доклада, после перерыва была ещё и [вторая](https://www.youtube.com/watch?v=cou_qomYLNU). Если по вашей реакции на этот хабрапост поймём, что такой контент нужен — возможно, сделаем текстовую версию и для второй части.

А пока что позовём вас на конференцию [**JPoint 2025**](https://jpoint.com/?utm_source=habr&utm_medium=893638), которая пройдет уже **3–4 апреля** (Москва \+ онлайн). Там тоже будет много контента для Java-разработчиков — и его можно смотреть без поправки на возраст доклада, всё свежее. Билеты давно в продаже, а спикеры уже на низком старте.

Закрыть

Теги:

* [доклад](/ru/search/?target_type=posts&order=relevance&q=[%D0%B4%D0%BE%D0%BA%D0%BB%D0%B0%D0%B4])
* [spring](/ru/search/?target_type=posts&order=relevance&q=[spring])
* [евгений борисов](/ru/search/?target_type=posts&order=relevance&q=[%D0%B5%D0%B2%D0%B3%D0%B5%D0%BD%D0%B8%D0%B9+%D0%B1%D0%BE%D1%80%D0%B8%D1%81%D0%BE%D0%B2])
* [spring-потрошитель](/ru/search/?target_type=posts&order=relevance&q=[spring-%D0%BF%D0%BE%D1%82%D1%80%D0%BE%D1%88%D0%B8%D1%82%D0%B5%D0%BB%D1%8C])

Хабы:

* [Блог компании JUG Ru Group](/ru/companies/jugru/articles/)
* [Java](/ru/hubs/java/)
* [Конференции](/ru/hubs/tech_events/)

Нравится

+22

Не нравится

Добавить в закладки90

[Комментарии4](/ru/companies/jugru/articles/893638/comments/)[+4](/ru/companies/jugru/articles/893638/comments/)