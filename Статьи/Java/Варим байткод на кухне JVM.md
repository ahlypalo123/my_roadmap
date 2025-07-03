---
title: "Варим байткод на кухне JVM"
source: "https://habr.com/ru/companies/domclick/articles/500646/"
author:
  - "[[AlexK23]]"
published: 2020-05-15
created: 2025-07-03
description: "Меня зовут Александр Коцюруба, я руковожу разработкой внутренних сервисов в компании ДомКлик. Многим разработчикам, пишущим на Java, с опытом приходит понимание внутреннего устройства JVM. Чтобы..."
tags:
  - "clippings"
---
Меня зовут Александр Коцюруба, я руковожу разработкой внутренних сервисов в компании ДомКлик. Многим разработчикам, пишущим на Java, с опытом приходит понимание внутреннего устройства JVM. Чтобы облегчить этот путь Java-самурая, я решил простым языком изложить основы виртуальной машины Java (JVM) и работы с байткодом.  
  
Что такое таинственный байткод и где он обитает?  
  
Постараюсь ответить на этот вопрос на примере приготовления солений.  
  
![](https://habrastorage.org/r/w1560/webt/e7/qk/0e/e7qk0ed8rejsgfy30aaosw8yrya.jpeg)  
  

## Зачем нужен JVM и байткод?

  
JVM возникла под лозунгом **Write Once Run Anywhere (WORA)** в стенах компании Sun Microsystems. В отличие от концепции **Write Once Compile Anywhere (WOCA)**, WORA подразумевает наличие виртуальной машины для каждой ОС, которая исполняет *единожды* скомпилированный код (байткод).  
  
![](https://habrastorage.org/r/w1560/webt/h_/lz/xv/h_lzxvvnwtozocqxyybcwgkypu8.png)  
*Write Once Run Anywhere (WORA)*  
  
![](https://habrastorage.org/r/w1560/webt/d-/ze/19/d-ze19v7ddfbstdjb8hyfuaq0s8.png)  
*Write Once Compile Anywhere (WOCA)*  
  
JVM и байткод лежат в основе концепции WORA и избавляют нас от нюансов и необходимости компиляции под каждую ОС.  
  

## Байткод

  
Чтобы понять, что из себя представляет байткод, давайте рассмотрим пример. Конечно, этот код не делает ничего полезного, он только послужит для дальнейшего разбора.  
  
Исходный код:  
  
```kotlin
class Solenya(val jarForPickles: Any? = Any(), var ingredientsCount: Int = 0) {

    /**
     *  Добавляет ингредиент
     *  @param ingredient - что добавляем
     */
    fun add(ingredient: Any) {
        ingredientsCount = ingredientsCount.inc()
        //какой-то код
    }

    /**
     *  Нагревает банку
     *  @param duration - сколько времени засекать
     */
    fun warmUp(duration: Int) {
        for (x in 1..duration)
            println("Warming")
    }

    init {
        //взять банку под соленья
        val jarForPickles = takeJarForPickles()
        //берем огурцы
        val pickles = Any()
        //берем воду
        val water = Any()

        //смешиваем
        add(pickles)
        add(water)

        //нагреваем
        warmUp(10)
    }

    /**
     *  Взять банку
     */
    private fun takeJarForPickles(): Any = openLocker()

    /**
     *  Открыть шкаф
     */
    private fun openLocker(): Any = takeKeyForLocker()

    /**
     *  Взять ключи под шкафом
     */
    private fun takeKeyForLocker(): Any = {}
}
```
  
С помощью встроенных инструментов Intellij IDEA (*Tools -> Kotlin -> Show Kotlin Bytecode*) получаем дизассемблированный байткод (в примере приведена лишь часть):  
  
```kotlin
...
   INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/Object;)V
   L5
   L6
    LINENUMBER 12 L6
    RETURN
   L7
    LOCALVARIABLE this Lcom/company/Solenya; L0 L7 0
    LOCALVARIABLE ingredient Ljava/lang/Object; L0 L7 1
    LOCALVARIABLE $i$f$add I L1 L7 2
    MAXSTACK = 2
    MAXLOCALS = 5

  // access flags 0x11
  public final warmUp(I)V
    // annotable parameter count: 1 (visible)
    // annotable parameter count: 1 (invisible)
   L0
    LINENUMBER 19 L0
    ICONST_1
    ISTORE 2
...
```
  
На первый взгляд — непонятный набор инструкций. Чтобы разобраться, как и с чем они работают, необходимо будет погрузиться во внутреннюю кухню JVM.  
  

## Кухня JVM

  
Посмотрим на JVM runtime memory:  
  
![](https://habrastorage.org/r/w1560/webt/oa/jh/6q/oajh6quzgdfk5evb5waoetjmmtc.png)  
  
Можно сказать, что JVM — наша кухня. Далее рассмотрим остальных участников:  
  

#### Method area — Кулинарная книга

  
![](https://habrastorage.org/r/w1560/webt/is/m-/v2/ism-v2gh1keewv8lsjhoni3g-vg.jpeg)  
В области Method area хранится скомпилированный код для каждой функции. Когда поток начинает выполнять функцию, в общем случае он получает инструкции из этой области. По сути, она представляет собой кулинарную книгу рецептов, где подробно описано, как приготовить всё подряд, начиная от яичницы и заканчивая каталонской сарсуэлой.  
  

#### Thread 1..N — Команда поваров

  
![](https://habrastorage.org/r/w1560/webt/x7/8b/qs/x78bqsqikxzixqylllxflpjm3fi.jpeg)  
Потоки строго выполняют предписанные им инструкции (method area), для этого у них есть PC Register и JVM Stack. Можно сравнить каждый поток с поваром, который выполняет данное ему поручение, в точности следуя рецептам из кулинарной книги.  
  

#### PC Register — Заметки на полях

  
![](https://habrastorage.org/r/w1560/webt/m1/tl/c2/m1tlc2rtzsubzyuebeigvvfotrm.png)  
Program Counter Register — счетчик команд нашего потока. Хранит в себе адрес выполняемой инструкции. На кухне это были бы некие заметки, на какой странице кулинарной книги мы сейчас находимся.  
  

#### JVM Stack

  
Стек фреймов. Под каждую функцию выделяется фрейм, в рамках которого текущий поток работает с переменными и операндами. В рамках аналогии с приготовлением наших солений это мог бы быть набор вложенных операций:  
  
`Приготовить соленья -> взять банку -> открыть шкаф -> взять ключи...`  
  

#### Frame — Рабочий стол

  
![](https://habrastorage.org/r/w1560/webt/ti/3k/fy/ti3kfyzgveqkh726jelzbedkpvk.jpeg)  
Фрейм выступает в роли рабочего стола повара, на котором лежит разделочная доска и подписанные контейнеры.  
  

#### Local variables — Подписанные контейнеры

  
![](https://habrastorage.org/r/w1560/webt/tl/as/rl/tlasrl-j3ocbkzd-irebk0usgsu.jpeg)  
Это массив локальных переменных (local variable table), который, как следует из названия, хранит значения, тип и область видимости локальных переменных. Это похоже на подписанные контейнеры, куда можно складывать промежуточные результаты профессиональной деятельности.  
  

#### Operand stack — Разделочная доска

  
![](https://habrastorage.org/r/w1560/webt/ic/90/m1/ic90m1h3sbynrbt-2kfyuixn0je.jpeg)  
Operand stack хранит аргументы для инструкций JVM. Например, целочисленные значения для операции сложения, ссылки на объекты heap и т. п.  
  
Самый близкий пример, который я могу привести — разделочная доска, на которой помидор и огурец в один момент превращаются в салат. В отличие от local variables на доску мы кладем только то, с чем будем выполнять ближайшую инструкцию.  
  

## Heap — Стол раздачи

  
![](https://habrastorage.org/r/w1560/webt/5o/gn/s5/5ogns5shcmsgcv8k5huixqrtw8q.jpeg)  
В рамках работы с фреймом мы оперируем ссылками на объекты, сами же объекты хранятся в heap. Важное отличие в том, что фрейм принадлежит только одному потоку, и локальные переменные «живут», пока жив фрейм (выполняется функция). А heap доступен и другим потокам, и живет до включения сборщика мусора. По аналогии с кухней, можно привести пример со столом раздачи, который один и является общим. И чистит его отдельная команда уборщиков.  
  

## JVM-кухня. Взгляд изнутри. Работа с Frame

  
Разберем для начала функцию `warmUp`:  
  
```kotlin
/**
     *  Нагревает банку
     *  @param duration - сколько времени засекать
     */
    fun warmUp(duration: Int) {
        for (x in 1..duration)
            println("Warming...")
    }
```
  
Дизассемблированный байткод функции:  
  
```kotlin
public final warmUp(I)V
    // annotable parameter count: 1 (visible)
    // annotable parameter count: 1 (invisible)
   L0
    LINENUMBER 19 L0
    ICONST_1
    ISTORE 2
    ILOAD 1
    ISTORE 3
    ILOAD 2
    ILOAD 3
    IF_ICMPGT L1
   L2
    LINENUMBER 20 L2
    LDC "Warming..."
    ASTORE 4
   L3
    ICONST_0
    ISTORE 5
   L4
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    ALOAD 4
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/Object;)V
   L5
   L6
    LINENUMBER 19 L6
    ILOAD 2
    ILOAD 3
    IF_ICMPEQ L1
    IINC 2 1
   L7
    GOTO L2
   L1
    LINENUMBER 21 L1
    RETURN
   L8
    LOCALVARIABLE x I L2 L7 2
    LOCALVARIABLE this Lcom/company/Solenya; L0 L8 0
    LOCALVARIABLE duration I L0 L8 1
    MAXSTACK = 2
    MAXLOCALS = 6
```
  

#### Инициализация фрейма — Подготовка рабочего места

  
Для выполнения этой функции в JVM stack потока будет создан frame. Напомню, что стек состоит из массива local variables и operand stack.  
  
1. Чтобы мы могли понять, сколько памяти выделять под данный фрейм, компилятор предоставил метаинформацию об этой функции (пояснения в комментарии к коду):  
	  
	```kotlin
	MAXSTACK = 2 // выделяем стек размером 2*32bit
	    MAXLOCALS = 6 // выделяем массив размером 6*32bit
	```
2. Также у нас есть информация о некоторых элементах массива local variable:  
	  
	```kotlin
	LOCALVARIABLE x I L2 L7 2 // переменная x типа Int(I), находится в области видимости меток L2-L7 под индексом 2
	    LOCALVARIABLE this Lcom/company/Solenya; L0 L8 0
	    LOCALVARIABLE duration I L0 L8 1
	```
3. Аргументы функции при инициализации фрейма попадают в local variables. В этом примере, значение duration будет записано в массив с индексом 1.
  
Таким образом, изначально фрейм будет выглядеть так:  
  
![](https://habrastorage.org/r/w1560/webt/5v/9t/7j/5v9t7jckxf046dd8ov-6yuuvaoq.png)  

#### Начало исполнения инструкций

  
Чтобы понять, как происходит работа с фреймом, достаточно вооружиться списком инструкций JVM ([Java bytecode instruction listings](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings)) и пошагово разобрать метку `L0`:  
  
```kotlin
L0
    LINENUMBER 19 L0 //метаинформация о соответствии строчки исходного кода
    ICONST_1
    ISTORE 2
    ILOAD 1
    ISTORE 3
    ILOAD 2
    ILOAD 3
    IF_ICMPGT L1
```
  
**ICONST\_1** — добавляем `1` (Int) в operand stack:  
  
![](https://habrastorage.org/r/w1560/webt/ha/e_/vj/hae_vjnnoyi5goccte90gebvap8.png)  
  
**ISTORE 2** — pull значения (с типом Int) из operand stack и запись в local variables с индексом 2:  
  
![](https://habrastorage.org/r/w1560/webt/ay/4p/oa/ay4poad-qt13zbsp15hbnxosou4.png)  
  
Эти две операции можно интерпретировать в Java-код: `int x = 1`.  
  
**ILOAD 1** — загрузить значение из local variables с индексом 1 в operand stack:  
  
![](https://habrastorage.org/r/w1560/webt/ut/1x/ul/ut1xulhulghlom_1l9yle_j4naq.png)  
  
**ISTORE 3** — pull значения (с типом Int) из operand stack и запись в local variables с индексом 3:  
  
![](https://habrastorage.org/r/w1560/webt/oz/hd/gz/ozhdgzi8r53br6gk8dxbnp06owi.png)  
  
Эти две операции можно интерпретировать в Java-код: `int var3 = duration`.  
  
**ILOAD 2** — загрузить значение из local variables с индексом 2 в operand stack.  
  
**ILOAD 3** — загрузить значение из local variables с индексом 3 в operand stack:  
  
![](https://habrastorage.org/r/w1560/webt/bt/sj/wj/btsjwjhugp0ett33ssfuaj5bjua.png)  
  
**IF\_ICMPGT L1** — инструкция сравнения двух целочисленных значений из стека. Если «нижнее» значение больше «верхнего», то переходим к метке `L1`. После выполнения этой инструкции стек станет пустым.  
  
Вот как выглядели бы эти строчки байткода на Java:  
  
```java
int x = 1;
      int var3 = duration;
      if (x > var3) {
         ....L1...
```
  
Декомпилируем кода с помощью Intellij IDEA по пути *Kotlin -> Java*:  
  
```java
public final void warmUp(int duration) {
      int x = 1;
      int var3 = duration;
      if (x <= duration) {
         while(true) {
            String var4 = "Warming";
            boolean var5 = false;
            System.out.println(var4);
            if (x == var3) {
               break;
            }
            ++x;
         }
      }
   }
```
  
Здесь можно увидеть неиспользуемые переменные (`var5`) и отсутствие вызова функции `println()`. Не стоит переживать, это связано со спецификой компиляции inline-функций (`println()`) и lambda-выражений. Накладных расходов на исполнение этих инструкций практически не будет, более того, dead code будет удален благодаря JIT. Это интересная тема, которой стоит посвятить отдельную статью.  
  
Проводя аналогию с кухней, эту функцию можно описать как задачу для повара «кипяти воду 10 минут». Далее наш профессионал своего дела:  
  
1. открывает кулинарную книгу (method area);
2. находит там инструкции, как кипятить воду (`warmUp()`);
3. готовит рабочее место, выделяя конфорку (operand stack) и контейнеры (local variables) для временного хранения продуктов.
  

## JVM-кухня. Взгляд изнутри. Работа с Heap

  
Рассмотрим код:  
  
```kotlin
val pickles = Any()
```
  
Дизассемблированный байткод:  
  
```kotlin
NEW java/lang/Object
    DUP
    INVOKESPECIAL java/lang/Object.<init> ()V
    ASTORE 3
```
  
**NEW java/lang/Object** — выделение памяти под объект класса `Object` из heap. В стек будет помещен не сам объект, а ссылка на него в heap:  
  
![](https://habrastorage.org/r/w1560/webt/5l/gg/p9/5lggp9fqms9rmhrvvgj5wzsn0s4.png)  
**DUP** — дублирование «верхнего» элемента стека. Одна ссылка нужна для иницализации объекта, вторая для ее сохранения в local variables:  
  
![](https://habrastorage.org/r/w1560/webt/ow/2u/fl/ow2uflabd3qw5iurha9jmltfvta.png)  
**INVOKESPECIAL java/lang/Object.<iniт> ()V** — инициализация объекта соответствующего класса (`Object`) по ссылке из стека:  
  
![](https://habrastorage.org/r/w1560/webt/3x/hm/h6/3xhmh6bsf0sgryvuxlujw7ve-we.png)  
**ASTORE 3** — последний шаг, сохранение ссылки на объект в local variables с индексом 3.  
  
Проводя аналогию с кухней, создание объекта класса я бы сравнил с приготовлением на общем столе (heap). Для этого необходимо выделить себе достаточно места на столе раздачи, вернуться на рабочее место и кинуть записку с адресом (reference) в соответствующий контейнер (local variables). И только после этого начать создавать объект класса.  
  

## JVM-кухня. Взгляд изнутри. Многопоточность

  
Теперь рассмотрим такой пример:  
  
```kotlin
fun add(ingredient: Any) {
        ingredientsCount = ingredientsCount.inc()
        //какой-то код
    }
```
  
Это классический пример проблемы многопоточности. У нас есть счетчик количества ингредиентов `ingredientsCount`. Функция `add`, помимо добавления ингредиента, выполняет инкремент `ingredientsCount`.  
  
Дизассемблированный байткод выглядит так:  
  
```kotlin
ALOAD 0
    ALOAD 0
    GETFIELD com/company/Solenya.ingredientsCount : I
    ICONST_1
    IADD
    PUTFIELD com/company/Solenya.ingredientsCount : I
```
  
Состояние нашего operand stack по ходу выполнения инструкций:  
  
![](https://habrastorage.org/r/w1560/webt/os/ng/c9/osngc9s9nkrgoo8dmx3foigyej8.png)  
При работе в один поток всё будет выполняться корректно. Если же потоков будет несколько, то может возникнуть следующая проблема. Представим, что оба потока одновременно получили значение поля `ingredientsCount` и записали его в стек. Тогда состояние operand stack и поля `ingredientsCount` может выглядеть так:  
  
![](https://habrastorage.org/r/w1560/webt/ne/oe/wh/neoewhd9o_qmry48pv9mqv6hrt4.png)  
Функция была выполнена дважды (по разу каждым потоком) и значение `ingredientsCount` должно бы быть равно 2. Но на деле один из потоков работал с устаревшим значением `ingredientsCount`, и поэтому фактический результат равен 1 (проблема Lost Update).  
  
Ситуация аналогична параллельной работе команды поваров, которые добавляют пряности в блюдо. Представим:  
  
1. Есть стол раздач, на котором лежит блюдо (Heap).
2. На кухне два повара (Thread\*2).
3. У каждого повара свой разделочный стол, где они готовят смесь пряностей (JVM Stack\*2).
4. Задача: добавить в блюдо две порции пряностей.
5. На столе раздач лежит бумажка, с которой читают и на которой пишут, какая по счету порция была добавлена (`ingredientsCount`). Причем в целях экономии пряностей:  
	- до начала подготовки пряностей повар должен прочитать на бумажке, что количество добавленных пряностей еще не достаточно;
	- после добавления пряностей повар может написать, сколько, по его мнению, пряностей добавлено в блюдо.
  
При таких условиях может возникнуть ситуация:  
  
1. Повар №1 прочитал, что было добавлено 3 порции пряностей.
2. Повар №2 прочитал, что было добавлено 3 порции пряностей.
3. Оба уходят к своим рабочим столам и готовят смесь пряностей.
4. Оба повара добавляют в блюдо пряности (3+2).
5. Повар №1 записывает, что было добавлено 4 порции пряностей.
6. Повар №2 записывает, что было добавлено 4 порции пряностей.
  
Итог: продуктов недосчитались, блюдо получилось острое и т.п.  
  
Чтобы избежать таких ситуаций, существуют разные инструменты вроде блокировок, thread-safety функций и т.д.  
  

## Подводя итоги

  
Крайне редко у разработчика возникает потребность лезть в байткод, если только это не является спецификой его работы. В то же время, понимание работы байткода помогает лучше понять многопоточность и преимущества того или иного языка, а также помогает дальше расти профессионально.  
  
Стоит отметить, что это далеко не все части JVM. Есть еще много интересных «штук», например, constant pool, bytecode verifier, JIT, code cache и т.д. Но чтобы не перегружать статью, я сосредоточился только на тех элементах, которые необходимы для общего понимания.  