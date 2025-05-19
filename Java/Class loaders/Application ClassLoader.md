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