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