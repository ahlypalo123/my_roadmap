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