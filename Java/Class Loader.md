Когда я впервые погрузился в мир загрузчиков классов Java, это было ответом на любопытный вопрос. Популярные источники ([Wikipedia](https://en.wikipedia.org/wiki/Java_Classloader), [Baeldung](https://www.baeldung.com/java-classloaders), [DZone](https://dzone.com/articles/java-virtual-machine-internals-class-loader)) содержат устаревшую, иногда противоречащую друг другу информацию, и это несоответствие послужило толчком для написания этой статьи — поиска ясности в лабиринте ClassLoader System.

Будучи разработчиком Java, вы наверняка сталкивались с `ClassNotFoundException` или `NoClassDefFoundError` — загадочными сообщениями, которые на мгновение останавливают наш процесс разработки. Класс не найден — понятно по названию, но не найден _где_? Кто и как его _ищет_, куда _доставляет_?

Попробуем погрузиться в эту тему вместе, отбросив сложности, в стиле небольших диаграмм. Полная картина того, о чем пойдет речь в этой серии статей:

![Система загрузчиков классов Java](https://habrastorage.org/r/w1560/getpro/habr/upload_files/eb5/76d/0f0/eb576d0f0ea8c56c71ad302afa08a37f.png "Система загрузчиков классов Java")

Система загрузчиков классов Java

### Предложение

Прежде чем перейти к рассмотрению механизмов работы загрузчиков классов, важно подчеркнуть одну деталь:

> Не существует _"универсальной"_ конструкции виртуальной машины Java.

[Спецификация JVM](https://docs.oracle.com/javase/specs/jvms/se20/html/) от компании Oracle, устанавливает ожидаемые компоненты и поведение для любой JVM. Однако, эта спецификация не предписывает конкретный подход к реализации этих компонентов, что приводит к тому, что на практике существует целый ряд уникальных реализаций, включая, но не ограничиваясь [HotSpot/OpenJDK](https://github.com/openjdk/jdk/tree/master/src/hotspot), [Eclipse OpenJ9](https://en.wikipedia.org/wiki/OpenJ9), [GraalVM](https://www.graalvm.org/) (основанной на OpenJDK). Каждая из реализаций следует спецификации, но при этом может отличаться по ряду аспектов, как производительность, стратегии сборки мусора и, как несложно предположить, детали процесса загрузки классов.

Отдельный момент, требующий внимания:

> Виртуальные машины Java _платформо-зависимы._

JVM для Windows OS не идентична JVM для Linux. "Но подождите", — скажете вы, — "я думал, что Java — это все о том, чтобы написать один раз, выполнить везде — независимость от платформы!". Совершенно верно. Однако независимость Java от платформы не означает, что JVM также независима от платформы. Совсем наоборот.

В большинстве статей на эту тему при описании не указывается ни конкретная версия Java, ни описанная реализация VM, что приводит к недопониманию, поскольку JVM развивается и изменяется с каждой версией. Сейчас лето 2023 года, и мир Java находится в предвкушении 21-й версии, но пока она не вышла, мы будем ориентироваться на **Java 20**, опираясь на саму [спецификацию JVM от Oracle](https://docs.oracle.com/javase/specs/jvms/se20/html/), и документацию [Oracle Java SE](https://docs.oracle.com/en/java/javase/20/docs/api/) для удобства.

Учитывая это, вернемся к нашей системе загрузчиков

---

## Начиная с основ

Говоря упрощенно, при запуске приложения **JVM загружает** в память необходимые классы, **проверяет** байткод, **выделяет** необходимые ресурсы и, наконец, **выполняет** код, преобразуя байткод в инструкции машинного языка, понятные конечной машине.

![Упрощенный путь преобразования исходного кода Java в нативный код платформы](https://habrastorage.org/r/w1560/getpro/habr/upload_files/e5e/6b9/7a1/e5e6b97a12ffe758ecd596e28164ca7a.png "Упрощенный путь преобразования исходного кода Java в нативный код платформы")

Упрощенный путь преобразования исходного кода Java в нативный код платформы

Но что на самом деле означает это _JVM загружает_? Спецификация Java SE [приводит](https://docs.oracle.com/javase/specs/jls/se20/html/jls-12.html#jls-12.2) следующий комментарий:

> _Loading refers to the process of finding the binary form of a class or interface with a particular name, perhaps by computing it on the fly, but more typically by retrieving a binary representation previously computed from source code by a Java compiler, and constructing, from that binary form, a_ `Class` _object to represent the class or interface._

Формулируя более простым языком, когда мы говорим о "загрузке класса", мы имеем в виду:

> Процесс **поиска** соответствующего файла .class на диске, **чтения** его содержимого и **передачи** его в среду выполнения JVM, которая представляет собой определенную часть памяти машины, предназначенную для выполнения вашего приложения.

## Погружаясь глубже

В действительности, система загрузчиков классов не просто находит классы — она _обеспечивает целостность и безопасность Java-приложения_, соблюдая правила бинарной структуры и пространства имен среды выполнения Java.

Стоит добавить, что она обеспечивает гибкость загрузки классов из различных источников — не только из локальной файловой системы, но и по сети, из базы данных или даже сгенерированных налету.

В этой статье мы углубимся в процесс загрузки, но для полного понимания стоит упомянуть, что этапа всего 3:

![Этапы системы загрузки классов Java](https://habrastorage.org/r/w1560/getpro/habr/upload_files/dfc/7c7/b1d/dfc7c7b1d8a3cdfe344a6f96f2d433eb.png "Этапы системы загрузки классов Java")

Этапы системы загрузки классов Java

### Загрузка (Loading) — Начальная фаза

Процесс начинается с того, что загрузчик класса (далее, _ClassLoader_) получает задание найти определенный класс, что может быть инициировано самой JVM, или вызвано командой в вашем коде. Задача же здесь заключается в том, чтобы _взять полное имя класса_ (например, `java.lang.String`) и _получить соответствующий файл класса_ (например, `String.class`) из его местоположения на диске —> в память JVM.

![Этап загрузки, начальный из этапов Системы Загрузчиков Классов Java](https://habrastorage.org/r/w1560/getpro/habr/upload_files/87d/d59/6bc/87dd596bcd35967d75508fad11fe202b.png "Этап загрузки, начальный из этапов Системы Загрузчиков Классов Java")

Этап загрузки, начальный из этапов Системы Загрузчиков Классов Java

Здесь важно понимать, что подсистема Загрузки — это не одиночный акт, а _иерархическая эстафета_. Каждый ClassLoader, родительский и дочерний, работает совместно, передавая эстафету ответственности до тех пор, пока нужный класс в конце концов не будет загружен.

Основополагающими принципами, определяющими этот скоординированный процесс загрузки классов, являются (полагайся на диаграмму для понимания):

- Видимость (_Visibility_): Дочерний ClassLoader может видеть классы, загруженные его родителем, но не наоборот, что обеспечивает инкапсуляцию;
    
- Уникальность (_Uniqueness):_ Класс, загруженный родителем, не будет повторно загружен его дочерним классом, что повышает эффективность;
    
- Иерархия делегирования (_Delegation Hierarchy_): Application ClassLoader (дочерний) передает запрос на загрузку класса родителям, загрузчикам Platform и Bootstrap. Если они не могут найти класс, то запрос передается обратно по цепочке, пока класс не будет найден, или не выкинут соответсвующий `ClassNotFoundException`
    

Рассмотрим каждый загрузчик подробнее.

#### Boostrap ClassLoader

![Bootstrap ClassLoader](https://habrastorage.org/r/w1560/getpro/habr/upload_files/ffb/2ab/b79/ffb2abb79b4aac7feed214f5a691645e.png "Bootstrap ClassLoader")

Bootstrap ClassLoader

Старейший представитель семейства, Bootstrap ClassLoader, отвечает за загрузку основных библиотек Java, расположенных в [java.base](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/module-summary.html) модуле (`java.lang`, `java.util` и т.д.), _необходимых для старта JVM_.

Обратя внимания на диаграмму можно заметить, что другие загрузчики классов написаны на Java (_объекты java.lang.ClassLoader_), что означает — их также необходимо загрузить в JVM! Эту задачу также выполняет Bootstrap ClassLoader.

> Во многих ресурсах Bootstrap ClassLoader описывается как "родитель" остальных загрузчиков классов. В действительность, это означает лишь _логическое наследование, а не наследование Java_, поскольку Bootstrap загрузчик написан на native коде, и встроен в виртуальную машину.

Убедимся на практике, что никаких Java загрузчиков, выше самих java.lang загрузчиков нет:

```sh
jshell> System.out.println(java.lang.ClassLoader.class.getClassLoader());
null
```

Bootstrap ClassLoader также является единственным загрузчиком, [_явно описанным в спецификации Oracle_](https://docs.oracle.com/javase/specs/jvms/se20/html/jvms-5.html#jvms-5.3.1). Остальные зовутся "User-defined", и оставляются на [_рассмотрение_](https://docs.oracle.com/javase/specs/jvms/se20/html/jvms-5.html#jvms-5.3.2) конкретных вендоров вирутальных машин.

### Platform ClassLoader

На мой взгляд, самый противоречивый.

![Platform ClassLoader](https://habrastorage.org/r/w1560/getpro/habr/upload_files/d97/c6c/b90/d97c6cb9069ce10d1c74416bec4b9037.png "Platform ClassLoader")

Platform ClassLoader

Документация Java SE 20 [говорит](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/ClassLoader.html#builtinLoaders) о нем следующее:

> The [platform class loader](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/ClassLoader.html#getPlatformClassLoader\(\)) is responsible for loading the _platform classes_. Platform classes include Java SE platform APIs, their implementation classes, and JDK-specific run-time classes that are defined by the platform class loader or its ancestors. The platform class loader can be used as the parent of a `ClassLoader` instance.

Но что отличает _классы платформы_ от _основных классов_, загружаемых Bootstrap загрузчиком? Посмотрим, что он на самом деле загружает:

```sh
jshell> ClassLoader.getPlatformClassLoader().getDefinedPackages();
$1 ==> Package[0] { } // empty
```

Получается, что в пустой Java-программе — абсолютно ничего! Теперь попробуем явно использовать класс из какого-нибудь стандартного пакета:

```sh
jshell> java.sql.Connection.class.getClassLoader()
$2 ==> jdk.internal.loader.ClassLoaders$PlatformClassLoader@27fa135a

jshell> ClassLoader.getPlatformClassLoader().getDefinedPackages()
$3 ==> Package[1] { package java.sql }
```

Получается, проще говоря, Bootstrap загружает основные классы **необходимые** для запуска JVM, а Platform — публичные типы системных модулей, которые **могут понадобиться**. Конкретного разделения необходимых/возможных модулей Java SE я не нашел, но задал соответсвующий вопрос на StackOverFlow, [ссылка](https://stackoverflow.com/questions/76699669/which-exact-classes-are-loaded-by-platform-classloader) для любознательных :)

В этом контексте также важно отметить, что во многих источниках ([Wiki](https://en.wikipedia.org/wiki/Java_Classloader), [Baeldung](https://www.baeldung.com/java-classloaders), последнее обновление 2022, 2023 соотсветственно) Platform ClassLoader обзывают _Extension ClassLoader_, что на деле не совсем так.

Правильнее было бы утверждать, что Platform ClassLoader _пришел на смену_ Extension ClassLoader, который искал в `$JAVA_HOME/lib/ext`, и использовался в Java 8 и более ранних версиях. Это изменение произошло с появлением [Системы Модулей (JEP-261)](https://openjdk.org/jeps/261):

> The extension class loader is no longer an instance of `URLClassLoader` but, rather, of an internal class. It no longer loads classes via the extension mechanism, which was removed by [JEP 220](http://openjdk.java.net/jeps/220). It does, however, define selected Java SE and JDK modules, about which more below. In its new role this loader is known as the platform class loader, it is available via the new`ClassLoader::getPlatformClassLoader` [method](http://download.java.net/java/jdk9/docs/api/java/lang/ClassLoader.html#getPlatformClassLoader--), and it will be required by the Java SE Platform API Specification.

#### Application ClassLoader

![Application (a.k.a. System) ClassLoader](https://habrastorage.org/r/w1560/getpro/habr/upload_files/144/9cd/759/1449cd759a6c3d25a539c3d84b4eb305.png "Application (a.k.a. System) ClassLoader")

Application (a.k.a. System) ClassLoader

Application ClassLoader, также известный как системный загрузчик классов, пожалуй, самый user-friendly из всех. Именно этот загрузчик подгружает **ваши собственные реализации** и **библиотеки зависимостей**, которые вы передали JVM (явно или неявно) при старте приложения в качестве `-classpath (-cp)` параметра.

```java
public class HabrTeller {

    public static void main(String[] args) {
        // jdk.internal.loader.ClassLoaders$AppClassLoader@251a69d7
        System.out.print(HabrTeller.class.getClassLoader());
    }
}
```

С точки зрения иерархии, Application загрузчик является порождением Platform загрузчика, и в документации о нем говорится следующее:

> This is the default delegation parent for new `java.lang.ClassLoader` instances, and is typically the class loader used to start the application.
> 
> `ClassLoader.getSystemClassLoader()` method is first invoked early in the runtime's startup sequence, at which point it creates the system class loader. This class loader will be the context class loader for the main applicati_on thread (for example, the thread that invokes the main method of the main class)._

Резюмирая, именно этот загрузчик является _родителем основного потока приложения_, и будет являться _родителем ваших собственных загрузчиков классов_, если вы решите реализовать один.

---

В дополнение к трем рассмотренным, основным загрузчикам, вы можете создавать свои собственные, пользовательские загрузчики классов, непосредственно в своих Java программах, позволяя обеспечить независимость приложений (чему способствует модель делегирования загрузчиков):

![Место пользовательского загрузчика классов в иерархии ClassLoader Loading System](https://habrastorage.org/r/w1560/getpro/habr/upload_files/80a/46e/5d6/80a46e5d6786d03812266de2608aeef4.png "Место пользовательского загрузчика классов в иерархии ClassLoader Loading System")

Место пользовательского загрузчика классов в иерархии ClassLoader Loading System

В серверах типа Tomcat, этот подход используется для обеспечения независимой работы различных Web-приложений и корпоративных решений, _даже если они размещены на одном сервере_. Из популярных открытых примеров, мне удалось найти несколько, для дополнительного ознакомления:

- [Tomcat's Catalina WebappLoader](https://github.com/apache/tomcat/blob/main/java/org/apache/catalina/loader/WebappLoader.java)
    
- [Spring Boot's LaunchedURLClassLoader](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-tools/spring-boot-loader/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java)
    

Почитать подробнее про обоснование создания собственных, и систему загрузчиков Tomcat как таковую, можно почитать [здесь](https://tomcat.apache.org/tomcat-9.0-doc/class-loader-howto.html).

Статей по созданию собственных загрузчиков классов написано уже немало, и целью этой статьи служит скорее теория, а не практика, но при должном интересе — можем написать обновленную, отдельную версию.

---

На этом этапе подпроцесс загрузки подходит к концу: результатом является двоичное представление класса или типа интерфейса в JVM. Однако на этом этапе класс еще не готов к использованию, и мы рассмотрим следующий этап — Linking — во второй части этой серии.

Спасибо, что дочитали до конца! Надеюсь, вы почерпнули что-то интересное. Данный материал не претендует на звание _single source of truth_, но мы действительно постарались ссылаться на официальную документацию и спецификацию языка, опуская субьективное и неофициальное.