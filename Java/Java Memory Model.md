---
title: "Глубокое погружение в Java Memory Model"
source: "https://habr.com/ru/articles/685518/"
author:
  - "[[Хабр]]"
published: 2022-08-30
created: 2025-04-27
description: "Я провел в изучении JMM много часов и теперь делюсь с вами знаниями в простой и понятной форме. В этой статье мы подробно разберем Java Memory Model (JMM) и применим полученные знания на практике. Да,..."
tags:
  - "clippings"
---
![](https://habrastorage.org/r/w1560/webt/eb/nm/iu/ebnmiu3gfh1rn74kqsnvqrv2rnm.png)

  

Я провел в изучении JMM много часов и теперь делюсь с вами знаниями в простой и понятной форме.

  

В этой статье мы подробно разберем Java Memory Model (JMM) и применим полученные знания на практике. Да, в интернете накопилось достаточно много информации про JMM/happens-before, и, кажется, что очередную статью про такую заезженную тему можно пропускать мимо. Однако я постараюсь дать вам намного большее и глубокое понимание JMM, чем большинство информации в интернете. После прочтения этой статьи вы будете уверенно рассуждать о таких вещах как memory ordering, data race и happens-before. JMM — сложная тема и не стоит верить мне на слово, поэтому большинство моих утверждений подтверждается цитатами из спеки, дизассемблером и jcstress тестами.

  

## Введение: контекст

  

В современном мире код часто выполняется не в том порядке, в котором он был написан в программе. Он часто переупорядочивается на уровне:

  
1. Компилятора байткода (в частности, *javac*)
2. Компилятора машинного кода (в частности, JIT компилятор *HotSpot C1/C2*). Например, среди компиляторов широко распространена такая оптимизация как [Instruction scheduling](https://en.wikipedia.org/wiki/Instruction_scheduling)
3. Процессора. Например, в мире процессоров широко распространены такие практики как *[Out-of-order execution](https://en.wikipedia.org/wiki/Out-of-order_execution)*, *[Branch Prediction](https://en.wikipedia.org/wiki/Branch_predictor) + [Speculation](https://en.wikipedia.org/wiki/Speculative_execution)*, *[Prefetching](https://en.wikipedia.org/wiki/Cache_prefetching)*, а также многие другие
  

Также в современных процессорах каждое ядро имеет собственный локальный кэш, который не видим другим ядрам. Более того, записи могут удерживаться в регистрах процессора, а не сбрасываться в память. Это ведет к тому, что поток может не видеть изменений, сделанных из других потоков.

  

Все эти оптимизации делаются с целью повысить производительность программ:

  
1. Переупорядочивание необходимо для того, чтобы найти самый оптимальный путь к выполнению кода, учитывая стоимость выполнения процессорных инструкций. Например, процессор может инициировать загрузку значения из памяти заранее, даже если в порядке программы это чтение идет позднее. Операции чтения из памяти стоят дорого, поэтому эта оптимизация позволяет максимально эффективно утилизировать процессор, избежав простаивания, когда это чтение действительно понадобится
2. Чтение из регистра и кэша стоит сильно дешевле, чем чтение из памяти. Более того, локальный кэш необходим для того, чтобы ядра не простаивали в ожидании доступа к общему кэшу, а могли работать с кэшем независимо друг от друга
  

Хорошо, но как в таком хаосе мы вообще можем написать корректную программу?

  

Есть хорошие новости, и плохие. Начнем с хорошей:

  
1. Java дает гарантию *[as-if-serial](https://en.wikipedia.org/wiki/As-if_rule)* выполнения кода — вне зависимости от используемой JDK итоговый результат выполнения будет не отличим от такого порядка, как если бы действия выполнялись действительно последовательно согласно порядку в коде
2. Процессоры тоже делают только такие переупорядочивания, которые не изменят итогового результата выполнения инструкций
3. Процессоры имеют Cache Coherence механизм, который гарантирует консистентность данных среди локальных кэшей: как только значение попадает в локальный кэш одного ядра, оно будет видно всем остальным ядрам
  

Рассмотрим на примере — этот однопоточный код может быть переупорядочен как угодно под капотом, но в итоге мы гарантированно увидим результат обеих записей при чтении:

  
```
a = 5;
b = 7;
int r1 = a; /* always 5 */
int r2 = b; /* always 7 */
```
  

Какой порядок инструкций мог быть под капотом?  
Например, такой:

  
```
b = 7;
a = 5;
int r2 = b; /* 7 */
int r1 = a; /* 5 */
```
  

Или такой:

  
```
b = 7;
int r2 = b; /* 7 */
a = 5;
int r1 = a; /* 5 */
```
  

Но здесь важно лишь то, что выполняемые под капотом действия в итоге приводят к ожидаемому результату. Такие переупорядочивания легальны потому, что эти 2 набора из записи/чтения никак не связаны друг с другом.

  

Теперь плохие новости:

  
1. Java дает *as-if-serial* гарантию только для единственного треда в изоляции. Это означает, что в многопоточной программе при работе с shared данными мы можем не увидеть записи там, где полагаемся на порядок выполнения действий в коде другого треда. Другими словами, для первого треда в изоляции валидно переупорядочивать инструкции местами, если это не повлияет на *его* результат выполнения, но переупорядочивание может повлиять на *другие* треды
2. Процессор также дает гарантию только для единственного ядра в изоляции
3. Cache Coherence действительно гарантирует чтение актуальных значений, но пропагация записи происходит не мгновенно, а с некоторой задержкой
  

Обо всем этом мы еще поговорим далее.

  

А теперь давайте перейдем к примеру из заголовка к статье (кстати, эта программа отражает идиому [Dekker lock](https://en.wikipedia.org/wiki/Dekker%27s_algorithm)):

  
```java
public class MemoryReorderingExample {
   private int x;
   private int y;

   public void T1() {
      x = 1;
      int r1 = y;
   }

   public void T2() {
      y = 1;
      int r2 = x;
   }
}
```
  

Проанализируем программу:

  
1. Обе записи идут до чтений, поэтому выполнение программы начинается или с записи x, или с записи y
2. Перед любым из чтений должна была произойти как минимум одна запись
  

Таким образом, кажется, что мы никогда не можем получить такой результат выполнения программы, когда увидим `0` на обоих чтениях. Иначе это означало бы, что выполнение программы началось с чтений, что не соответствует порядку программы.  
Однако, хоть это и может показаться странным, в данной программе мы вполне можем наблюдать результат чтения `(r1, r2) = (0, 0)`. А причины следующие:

  
1. *Instructions reordering*. Оба треда могли поменять местами инструкции записи и чтения, так как эти действия никак не связаны
2. *Visibility*. Даже если переупорядочивания не было, записи могут быть просто не видны другому треду из-за оптимизаций компилятора или задержки при пропагации записи на уровне кеша
  

Совсем не нужно верить мне на слово, поэтому давайте напишем тест при помощи инструмента [jcstress](https://github.com/openjdk/jcstress), который позволяет писать concurrency тесты для Java:

  
```java
@JCStressTest
@Description("Classic test that demonstrates memory reordering")
@Outcome(id = "1, 1", expect = Expect.ACCEPTABLE, desc = "Have seen both writes")
@Outcome(id = {"0, 1", "1, 0"}, expect = Expect.ACCEPTABLE, desc = "Have seen one of the writes")
@Outcome(id = "0, 0", expect = Expect.ACCEPTABLE_INTERESTING, desc = "Have not seen any write")
public class JmmReorderingDekkerTest {

   @Actor
   public final void actor1(DataHolder dataHolder, II_Result r) {
      r.r1 = dataHolder.actor1();
   }

   @Actor
   public final void actor2(DataHolder dataHolder, II_Result r) {
      r.r2 = dataHolder.actor2();
   }

   @State
   public static class DataHolder {
      private int x;
      private int y;

      public int actor1() {
         x = 1;
         return y;
      }

      public int actor2() {
         y = 1;
         return x;
      }
   }
}
```
  

Вот как нужно интерпретировать результат теста:

  
- `(1, 1): Expect.ACCEPTABLE` — мы прочитали обе записи. Это корректное поведение
- `(0, 1), (1, 0): Expect.ACCEPTABLE` — мы прочитали одно из значений слишком рано. Это корректное поведение
- `(0, 0): Expect.ACCEPTABLE_INTERESTING` — мы не увидели ни одной записи. Это случай instructions reordering/visibility
  

Запускаем тест на Intel Core i7-11700 (x86), Windows 10 x64, OpenJDK 17 с помощью команды `java -jar jcstress.jar -t JmmReorderingDekkerTest -v -f 10` (подробную инструкцию по сборке и запуску тестов вы сможете найти в моем репозитории, который я приведу в конце статьи):

  
```
RESULT         SAMPLES     FREQ       EXPECT  DESCRIPTION
  0, 0   2,188,517,311   18,91%  Interesting  Have not seen any write
  0, 1   4,671,980,718   40,36%   Acceptable  Have seen one of the writes
  1, 0   4,708,890,866   40,68%   Acceptable  Have seen one of the writes
  1, 1       5,569,185    0,05%   Acceptable  Have seen both writes
```
  

Как видите, в 18,91% случаев от общего количества прогонов мы не увидели ни одной записи.

  

Все еще не убедительно? А давайте заглянем на самый нижний уровень — уровень JIT-сгенированного кода. Напишем такую тестовую программу:

  
```java
public class JmmDekkerInstructionsReorderingJIT {

    private static int x;
    private static int y;

    private static int T1(int i) {
        x = i;
        return y;
    }

    private static int T2(int i) {
        y = i;
        return x;
    }

    public static void main(String[] args) throws Exception {
        // invoke JIT
        for (int i = 0; i < 10000; i++) {
            T1(i);
            T2(i);
        }
        Thread.sleep(1000);
    }
}
```
  

Теперь возьмем дизассемблер [hsdis](https://blogs.oracle.com/javamagazine/post/java-hotspot-hsdis-disassembler) и посмотрим на сгенерированный JIT-компилятором нативный код (подробная инструкция по самостоятельному запуску также будет в моем репозитории). Запускаем дизассемблер на Intel Core i7-11700 (x86), Windows 10 x64, OpenJDK 17 с помощью команды `java -server -XX:+UnlockDiagnosticVMOptions -XX:+StressLCM -XX:+StressGCM -XX:-TieredCompilation -XX:+PrintAssembly -XX:PrintAssemblyOptions=intel JmmDekkerInstructionsReorderingJIT`. Вот сгенерированный ASM код для обоих методов:

  
```
[Verified Entry Point]
  # {method} {0x0000015b92003208} 'T1' '(I)I' in 'jit_disassembly/JmmDekkerInstructionsReorderingJIT'
  # parm0:    rdx       = int
  #           [sp+0x20]  (sp of caller)
  0x0000015bf9561c00:   sub    rsp,0x18
  0x0000015bf9561c07:   mov    QWORD PTR [rsp+0x10],rbp     ;*synchronization entry
                                                            ; - jit_disassembly.JmmDekkerInstructionsReorderingJIT::T1@-1 (line 9)
  0x0000015bf9561c0c:   movabs r10,0x7117dd288              ;   {oop(a 'java/lang/Class'{0x00000007117dd288} = 'jit_disassembly/JmmDekkerInstructionsReorderingJIT')}
  0x0000015bf9561c16:   mov    eax,DWORD PTR [r10+0x74]     ;*getstatic y {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.JmmDekkerInstructionsReorderingJIT::T1@4 (line 10)
  0x0000015bf9561c1a:   mov    DWORD PTR [r10+0x70],edx     ;*putstatic x {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.JmmDekkerInstructionsReorderingJIT::T1@1 (line 9)

[Verified Entry Point]
  # {method} {0x0000015b920032b0} 'T2' '(I)I' in 'jit_disassembly/JmmDekkerInstructionsReorderingJIT'
  # parm0:    rdx       = int
  #           [sp+0x20]  (sp of caller)
  0x0000015bf9561f00:   sub    rsp,0x18
  0x0000015bf9561f07:   mov    QWORD PTR [rsp+0x10],rbp     ;*synchronization entry
                                                            ; - jit_disassembly.JmmDekkerInstructionsReorderingJIT::T2@-1 (line 14)
  0x0000015bf9561f0c:   movabs r10,0x7117dd288              ;   {oop(a 'java/lang/Class'{0x00000007117dd288} = 'jit_disassembly/JmmDekkerInstructionsReorderingJIT')}
  0x0000015bf9561f16:   mov    eax,DWORD PTR [r10+0x70]     ;*getstatic x {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.JmmDekkerInstructionsReorderingJIT::T2@4 (line 15)
  0x0000015bf9561f1a:   mov    DWORD PTR [r10+0x74],edx     ;*putstatic y {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.JmmDekkerInstructionsReorderingJIT::T2@1 (line 14)
```
  

Как видите, запись и чтение были переупорядочены друг с другом. Стало страшно? Читайте далее, чтобы не попасть в такую ситуацию.

  

---

  

## Введение: JMM

  

Теперь, получив контекст и поняв проблемы, можно начать говорить о JMM.

  

Мы поняли, что as-if-serial семантики недостаточно для многопоточных программ. Почему же не распространить as-if-serial гарантию на всю программу и ядра процессора? Ответ простой — это сильно ударило бы по производительности программ или процессора.

  

Одно из решений описанных проблем — это начать полагаться на строгие гарантии определенной микро-архитектуры процессора или имплементации компилятора/JVM. Но это очень хрупкое решение, которое заставляет думать о среде запуска программы, что препятствует кросс-платформенности. Например, ARM архитектура обладает гораздо более слабыми гарантиями по сравнению с x86: мы можем обнаружить намного больше багов в программе, если однажды стабильно работавшую на x86 программу запустим на ARM. Более того, обычно компиляторы не дают никаких гарантий, а вольны делать любые оптимизации.

  

В общем, нам нужна поддержка со стороны спецификации языка. Поэтому более надежное решение — это создание так называемой *модели памяти* (memory model), которая строго описывает какое выполнение программы является валидным. Модель памяти делает легальными многие оптимизации компилятора, JVM и процессора, но в то же время закрепляет условия, при которых программа будет вести себя корректно в многопоточной среде даже в присутствие оптимизаций. Таким образом, модель памяти:

  
- Разрешает выполнение различных оптимизаций компилятора, JVM или процессора
- Строго закрепляет условия, при которых программа считается правильно синхронизированной, и закрепляет поведение правильно синхронизированных программ
- Описывает отношение между высокоуровневым кодом и памятью
- Является trade-off между строгостью исполнения кода и возможными оптимизациями
  

Так вот, Java имеет свою модель памяти под названием **Java Memory Model** (JMM). По умолчанию JMM разрешает любые переупорядочивания и не гарантирует видимости изменений. Однако *при выполнении определенных условий* нам гарантируется порядок действий, консистентный с порядком в коде, а также видимость всех изменений. Таким образом, JMM позволяет нам писать программы, которые будут полностью корректно работать среди множества различных имплементаций JDK и микро-архитектур процессоров, в то же время сохраняя преимущества оптимизаций.

  

---

  

## Введение: Memory Ordering

  

Для полного понимания модели памяти нам необходимо разобрать такое понятие как "memory ordering".

  

**Memory Ordering** описывает *наблюдаемый программой* порядок, в котором происходят действия с памятью.

  

Смотрите: со стороны программы есть только действия записи/чтения и их порядок в коде. Также со стороны программы кажется, что мы имеем единую общую память, записи в которую становятся сразу видны другим тредам. Программа не подозревает ни о каких instruction scheduling reordering/out-of-order execution/caching/register allocation и прочих оптимизациях под капотом. Если по какой-то причине мы наблюдаем результат, не консистентный с порядком в программе, то со стороны программы (высокоуровнево) это выглядит так, что действия c памятью просто были переупорядочены. Другими словами, *порядок взаимодействия с памятью (memory order) может отличаться от порядка действий в коде (program order)*.

  

Для большего понимания давайте взглянем на уже знакомую нам программу с точки зрения Memory Ordering:

  

| Thread 0 | Thread 1 |
| --- | --- |
| x = 1 | y = 1 |
| r1 = y | r2 = x |

  

В случае результата выполнения `(r1, r2) = (0, 0)` мы можем просто сказать, что *произошел `StoreLoad` memory reordering*, то есть запись произошла после чтения. Не важно, по какой низкоуровневой причине это случилось, а важно лишь то, что в итоге со стороны программы действия с памятью были выполнены в неконсистентном порядке.

  

Таким образом, в многопоточной программе нам важно знать ответы на следующие вопросы:

  
1. **Как сохраняется порядок программы при работе с памятью?**
2. **Валиден ли наблюдаемый memory order?**
  

Дать ответ на каждый из вопросов — это и есть задача модели памяти. Java Memory Model разрешает все возможные переупорядочивания в отсутствие синхронизации, поэтому ответ на эти вопросы такой:

  
1. Если программа не синхронизирована, то разрешены все переупорядочивания. Если программа правильно синхронизирована, запрещены все переупорядочивания
2. Если программа не синхронизирована, то memory order, неконсистентный с program order, валиден с точки зрения JMM. Если программа правильно синхронизирована, то валиден *только* консистентный порядок
  

Ваша программа отрабатывает в одном из порядков, валидных с точки зрения JMM. Таким образом, если программа не правильно синхронизирована, не стоит удивляться некорретному результату выполнения. Ведь важно то, валиден ли результат выполнения с точки зрения модели памяти, а не то, валиден он или нет для вас как пользователя.

  

Однако то, что какой-то неконсистентный порядок валиден, еще не значит, что вы всегда получите некорректный результат, ведь и консистентный порядок возможен в отсутствие синхронизации — это вы могли видеть по jcstress тесту, который является вероятностным. Понятно, что вы не хотите надеяться на волю случая, поэтому необходимо ограничить возможный сет порядков выполнения до только консистентных. А для этого необходимо использовать предоставляемые моделью примитивы синхронизации, которые мы рассмотрим позднее.

  

В свою очередь, *Memory Reordering* — это высокоуровневое понятие, которое абстрагирует и обобщает низкоуровневые проблемы, которые мы рассматривали выше. Всего существует 4 типа memory reordering:

  
1. *LoadLoad*: переупорядочивание чтений с другими чтениями. Например, действия `r1, r2` могут выполниться в порядке `r2, r1`
2. *LoadStore*: переупорядочивание чтений с записями, идущими позже в порядке программы. Например, действия `r, w` могут выполниться в порядке `w, r`
3. *StoreStore*: переупорядочивание записей с другими записями. Например, действия `w1, w2` могут выполниться в порядке `w2, w1`
4. *StoreLoad*: переупорядочивание записей с чтениями, идущими позже в порядке программы. Например, действия `w, r` могут выполниться в порядке `r, w`
  

В дальнейшем, когда я буду говорить "переупорядочивание" или "reordering", я буду иметь в виду именно Memory Reordering, если не сказано обратное.

  

*Memory Model* описывает, какие переупорядочивания возможны. В зависимости от строгости модели памяти подразделяются на следующие типы:

  
1. *Sequential Consistency*: запрещены все переупорядочивания
2. *Relaxed Consistency*: разрешены некоторые переупорядочивания
3. *Weak Consistency*: разрешены все переупорядочивания
  

Модель памяти существует как на уровне языка, так и на уровне процессора, но они не связаны напрямую. Модель языка может предоставлять как более слабые, так и более строгие гарантии, чем модель процессора.

  

В частности, как уже было сказано выше, Java Memory Model не дает никаких гарантий, пока не использованы необходимые примитивы синхронизации. И напротив, посмотрите на главу Memory Ordering из [Intel Software Developer’s Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf#G13.14501):

  

> - Reads are not reordered with other reads \[запрещает LoadLoad reordering\]
> - Writes are not reordered with older reads \[запрещает LoadStore reordering\]
> - Writes to memory are not reordered with other writes \[запрещает StoreStore reordering\]
> - Reads may be reordered with older writes to different locations but not with older writes to the same location \[разрешает StoreLoad reordering\]
>   

Как видите, Intel разрешает только `StoreLoad` переупорядочивания, а все остальные запрещены. Да, модель памяти x86 достаточно строга, но есть и намного более слабые модели памяти процессоров — например, ARM разрешает все переупорядочивания.

  

Однако даже если вы пишете программу под x86, вам все равно необходимо считаться с более слабой Java Memory Model, так как последняя разрешает все переупорядочивания на уровне компилятора. *Модель памяти языка — прежде всего.*

  

---

  

### Memory Ordering vs Instructions Ordering

  

Еще раз закрепим: Memory Ordering и Instructions Ordering — это не одно и то же. Инструкции могут переупорядочиваться под капотом как угодно, но их *memory effect* должен подчиняться некоторым Memory Ordering правилам, которые гарантируются (или не гарантируются) Memory Model. Наконец, *memory ordering* — это высокоуровневое понятие, созданное для простоты понимания работы с памятью.

  

Например, Intel запрещает `LoadLoad` переупорядочивания, но под капотом все равно делает спекулятивные чтения. Как это возможно? Дело в том, что процессор следит за тем, чтобы результат выполнения инструкций не нарушал memory ordering правил. Если какое-то правило нарушается, то процессор возвращается к более раннему состоянию: результат чтения отбрасывается, а записи не коммитятся в память. Например, из того же [Intel Software Developer’s Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf#G13.31870):

  

> The processor-ordering model described in this section is virtually identical to that used by the Pentium and Intel486 processors. The only enhancements in the Pentium 4, Intel Xeon, and P6 family processors are:  
> - Added support for speculative reads, while still adhering to the ordering principles above.
>   

  

---

  

## Введение: Sequential Consistency

  

**Sequential Consistency Model (SC)** — это очень строгая модель памяти, которая гарантирует отсутствие переупорядочиваний.

  

Интуитивно SC можно понять очень просто: возьмите действия тредов, как они идут в порядке программы, и просто выполните их последовательно, возможно переключаясь между тредами.

  

Формальное определение SC также достаточно простое:

  

> \[Lamport, 1979 — [How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs](https://lamport.azurewebsites.net/pubs/multi.pdf)\]...the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

Давайте разберем SC на примере. Возьмем все тот же Dekker lock, который мы рассматривали выше:

  

| Thread 0 | Thread 1 |
| --- | --- |
| x = 1 | y = 1 |
| r1 = y | r2 = x |

  

В SC модели могут быть следующие memory order и никакие больше:

  
```
write(x, 1) -> write(y, 1) -> read(y):1 -> read(x):1
write(x, 1) -> write(y, 1) -> read(x):1 -> read(y):1
write(x, 1) -> read(y):0 -> write(y, 1) -> read(x):1

write(y, 1) -> write(x, 1) -> read(x):1 -> read(y):1
write(y, 1) -> write(x, 1) -> read(y):1 -> read(x):1
write(y, 1) -> read(x):0 -> write(x, 1) -> read(y):1
```
  

Назовем такие порядки *"sequentially consistent memory orders"*.

  

А вот такой memory order, где присутствуют `StoreLoad` переупорядочивания и которые дают нам результат `(r1, r2) = (0, 0)`, запрещен в SC:

  
```
read(y):0 -> read(x):0 -> write(x, 1) -> write(y, 1)
```
  

---

  

## Введение: Sequential Consistency-Data Race Free

  

Отлично, все это звучит здорово, но как же нам получить такую модель памяти? Ведь как мы уже поняли, JMM — это слабая модель памяти, которая не гарантирует консистентного порядка памяти.

  

Однако я уже упоминал выше, что при соблюдении некоторых условий наша программа будет считаться правильно синхронизированной и всегда работать корректно. Так вот, Java Memory Model — это **Sequential Consistency-Data Race Free (SC-DRF)** модель: нам предоставляется sequential consistency, но только в том случае, если мы избавимся от всех *data race* в программе — про это мы еще поговорим далее.

  

---

  

## Введение: Sequential Consistency: Why?

  

Вы наверное сейчас сидите и думаете: абстракция над абстракцией и абстракцией погоняет… Memory model, memory order, sequential consistency… Ну зачем, зачем же все эти абстракции? Давайте вместе разбираться.

  

Посмотрим на определение Sequential Consistency еще раз:

  

> ...the result of any execution **is the same as if** the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

Обратите внимание на выделенное в определении *"is the same as if"* — это очень важная деталь. Это означает, что инструкции под капотом не должны выполняться именно в sequential порядке. Важно лишь то, чтобы результат выполнения был *не отличим* от одного из таких порядков.

  

Хорошо, а нам то что с этого? А нам ничего. Зато разработчикам JMM такое определение позволяет делать любые оптимизации под капотом, пока они не приводят к результату, который не возможен в SC выполнении.

  

Давайте разберем подробнее. Снова возвращаемся к нашей программе:

  

| Thread 0 | Thread 1 |
| --- | --- |
| x = 1 | y = 1 |
| r1 = y | r2 = x |

  

Снова перечисляем все возможные sequentially consistent порядки, где каждому выполнению запишем конечный результат:

  
```
write(x, 1) -> write(y, 1) -> read(y):1 -> read(x):1 // result: (x,y)=(1, 1)
write(x, 1) -> write(y, 1) -> read(x):1 -> read(y):1 // result: (x,y)=(1, 1)
write(x, 1) -> read(y):0 -> write(y, 1) -> read(x):1 // result: (x,y)=(1, 0)

write(y, 1) -> write(x, 1) -> read(x):1 -> read(y):1 // result: (x,y)=(1, 1)
write(y, 1) -> write(x, 1) -> read(y):1 -> read(x):1 // result: (x,y)=(1, 1)
write(y, 1) -> read(x):0 -> write(x, 1) -> read(y):1 // result: (x,y)=(0, 1)
```
  

В итоге мы получаем следующий сет возможных результатов: `(0, 1), (1, 0), (1, 1)`. Назовем такие результаты *"sequentially consistent results"*.

  

Так вот, любая имплементация нашей программы, которая гарантированно приводит *к одному из* sequentially consistent результатов, является валидной вне зависимости от того, какие оптимизации были сделаны. Главное то, чтобы эти оптимизации не могли привести к sequentially inconsistent результату.

  

Например, в SC модели памяти компилятор не может вот так переставить чтение с записью в нашей программе:

  

| Thread 0 | Thread 1 |
| --- | --- |
| r1 = y | y = 1 |
| x = 1 | r2 = x |

  

Ведь такая имплементация может привести к результату `(r1, r2)=(0, 0)`:

  

| Thread 0 | Thread 1 |
| --- | --- |
| r1 = y // 0 |  |
|  | y = 1 |
|  | r2 = x // 0 |
| x = 1 |  |

  

А поэтому компилятору запрещается делать такую оптимизацию, ведь такой результат не входит в набор sequentially consistent результатов, который мы определили выше.

  

А вот, например, какую оптимизацию компилятор может сделать:

  

| Thread 0 | Thread 1 |
| --- | --- |
| r1 = 1 | r2 = 1 |

  

Компилятор полностью убрал записи и просто заинлайнил значения в чтения. Нарушает ли это Sequential Consistency? Совсем нет, ведь эта имплементация всегда приводит нас к результату `(r1, r2)=(1, 1)`, который является *sequentially consistent* результатом. То есть, если существует как минимум один sequentially consistent порядок, который дает такой результат, то такая оптимизация валидна. Конечно, это очень простой и наивный пример. Но расширьте обзор до всей программы и всех умных оптимизаций компилятора и JVM, и вы получите высоко-производительную программу, которая все равно выполняется полностью корректно.

  

Таким образом, представить себе работу SC модели памяти можно следующим образом:

  
1. Модель памяти "анализирует" оригинальную программу и исходя из порядка действий в программе "просчитывает" сет всех возможных sequentially consistent порядков и их результатов
2. С учетом получившегося сета результатов модель памяти разрешает делать любые оптимизации, пока они приводят к одному из sequentially consistent результатов, и запрещает делать такие оптимизации, которые могут привести к sequentially inconsistent результату
  

Что получается в итоге? А в итоге модель памяти, memory order, sequential consistency — это все абстракция между нами, пользователями JMM, и собственно самой имплементацией JMM. Нам данные абстракции позволяют нам рассуждать о корректности нашей программы без вдавания в низкоуровневые подробности, а разработчикам JMM делать любые оптимизации под капотом, пока они не нарушают этих абстракций. Однако в рамках данной статьи я все-таки покрою уровни ниже, чтобы иметь полную картину. В конце концов, как можно понять важность JMM, если не увидеть хоть части от всей сложности под капотом?

  

So, here's the deal: вы избавляетесь от всех *data race* в коде и получаете программу, результат которой будет всегда не отличим от одного из sequentially consistent порядков, а разработчики JMM получают возможность делать любые оптимизации под капотом, пока они приводят к валидному результату.

  

---

  

## Введение: data race

  

*Data race* возникает тогда, когда с shared данными работает одновременно два или больше тредов, где как минимум один из них пишет и их действия *не синхронизированы*. Для действий в гонке не гарантируется никакого консистентного memory order, поэтому не стоит удивляться неожиданным результатам.

  

*Data race* в рамках JMM — это ключевая вещь, которая формально отличает SC и не-SC выполнения: если мы докажем, что никакое выполнение нашей программы не имеет гонок, то результат выполнения программы будет всегда объясним с точки зрения одного из sequentially consistent порядков.

  

Давайте пройдемся по формальным определениям в спеке.

  

Для начала взглянем на формальное определение data race:

  
1. [JLS §17.4.1. Shared Variables](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4.1):  
	> Memory that can be shared between threads is called *shared memory* or *heap memory*.  
	>   
	> All instance fields, `static` fields, and array elements are stored in heap memory. In this chapter, we use the term *variable* to refer to both fields and array elements.  
	>   
	> Two accesses to (reads of or writes to) the same variable are said to be *conflicting* if at least one of the accesses is a write.
2. [JLS §17.4.5. Happens-before Order](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4.5):  
	> When a program contains two *conflicting accesses* (§17.4.1) that are not ordered by a *happens-before relationship*, it is said to contain a **data race**.
  

А теперь найдем ответ на следующий вопрос: *как же нам добиться SC?* Смотрим на [JLS §17.4.3. Programs and Program Order](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4.3):

  

> A set of actions is *sequentially consistent* if all actions occur in a total order (the execution order) that is consistent with program order, and furthermore, each read r of a variable v sees the value written by the write w to v such that:  
> - w comes before r in the execution order, and
> - there is no other write w' such that w comes before w' and w' comes before r in the execution order.
>   
>   
> Sequential consistency is a very strong guarantee that is made about visibility and ordering in an execution of a program. Within a sequentially consistent execution, there is a total order over all individual actions (such as reads and writes) which is consistent with the order of the program, and each individual action is atomic and is immediately visible to every thread.  
>   
> **If a program has no data races, then all executions of the program will appear to be sequentially consistent.**

Вот и то самое *SC-DRF*, про которое мы говорили выше: чтобы добиться sequential consistency, необходимо избавиться от всех data race в программе. Все это звучит просто, но не так просто это сделать.

  

Избавиться от гонок можно двумя способами:

  
1. Связать все действия с shared данными в synchronization order
2. Связать все действия с shared данными в happens-before order
  

Далее мы рассмотрим оба этих способа. Ну что ж, поехали!

  

---

  

## JMM: Synchronization Order

  

Как мы уже знаем, JMM — это слабая модель памяти, которая не соблюдает порядок программы. Поэтому модель должна нам предоставить специальные примитивы, при использовании которых гарантировался бы консистентный порядок.

  

И такое решение действительно есть: JMM предоставляет специальные примитивы под названием *Synchronization Actions (SA)*. Для таких действий образуется полный порядок под названием *Synchronization Order (SO)*, в котором:

  
1. Порядок действий *консистентен с порядком программы*
2. Каждое чтение всегда видит предыдущую запись
  

[JLS 17.4.4. Synchronization Order](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4.4):

  

> Every execution has a *synchronization order*. A synchronization order is a total order over all of the synchronization actions of an execution. For each thread `t`, the synchronization order of the synchronization actions (§17.4.2) in `t` is consistent with the program order (§17.4.3) of `t`.

Постойте, но это же звучит как Sequential Consistency? Именно так — SO это SC!

  

Нас интересуют следующие synchronization actions:

  
1. Запись и чтение `volatile` переменной
2. Взятие и освобождение монитора
  

Давайте сразу перейдем к практике. Например, если в Dekker lock мы пометим обе переменные как `volatile`, то свяжем все действия с переменными в SO и получим только SC выполнения:

  
```java
public class DekkerVolatile {
   private volatile int x;
   private volatile int y;

   public void T1() {
      x = 1;
      int r1 = y;
   }

   public void T2() {
      y = 1;
      int r2 = x;
   }
}
```
  

Ранее мы уже перечисляли все возможные SC порядки, поэтому не будем их повторять. Давайте лучше для нагладяности перечислим порядки, которые *не подчиняются* свойствам SO, а поэтому не считаются валидными с точки зрения JMM:

  
```
write(x, 1) --SO-> read(y):0 --non-SO-> read(x):1 --non-SO-> write(y, 1) // violates SO-PO consistency - not consistent with program order
read(y):0 --non-SO-> read(x):0 --non-SO-> write(x, 1) --non-SO-> write(y, 1) // violates SO-PO consistency - not consistent with program order
write(x, 1) --SO-> write(y, 1) -> read(y):0 --SO-> read(x):0 // violates SO consistency - does not see preceding writes
```
  

Итак, мы получили Sequential Consistency, и на этом разговор о JMM можно завершать. Всем спасибо за внимание.

  

Шучу. Заметим, что SO предлагает нам все или ничего: или мы используем только synchronization actions (например, помечаем все shared переменные как volatile) и получаем SC, или не получаем ничего, ведь не-SA действия (обычные записи и чтения shared переменной) снова будут выполняться в неконсистентном порядке, нарушая корректность программы.

  

Более того, синхронизировать всю программу таким способом просто неудобно: представьте, что вы пишете объект с кучей полей — вам придется пометить как внутренние поля, так и ссылку на объект как `volatile`. Иначе вы рискуете получить такое StoreStore переупорядочивание, что запись ссылки произойдет до записей из конструктора (как минимум для процессора это все одно и то же, поэтому он способен сделать такое переупорядочивание).

  

Давайте теперь зададимся вопросом: зачем вообще JMM явно предоставляет SA, если можно было бы сделать все действия такими неявно и дать нам SC по-умолчанию? Сиди и помечай все переменные как `volatile`, нам ведь заняться нечем.

  

Хотя связать все действия с shared данными в SO иногда бывает необходимо (как в Dekker локе или [IRIW](https://github.com/openjdk/jcstress/blob/master/tests-custom/src/main/java/org/openjdk/jcstress/tests/volatiles/VolatileIRIWTest.java) тесте), но в большинстве случаев это избыточно и запретит многие оптимизации компилятора, ухудшив производительность программы. Да, SO/SC действительно представляют собой лишь иллюзию последовательного выполнения, но многие оптимизации все равно будут запрещены. Именно поэтому ни в каком языке или микроархитектуре мы никогда не встретим SC модели по-умолчанию.

  

Так вот, в JMM существует основанный на SA формальный способ разрешить оптимизации под капотом, при этом все так же дав пользователям возможность писать корректные программы. И вот наконец мы начинаем говорить о частичном *happens-before (hb)* порядке, который все так же гарантирует консистентный порядок и видимость изменений, но при этом является намного более удобным в использовании и позволяет нам не синхронизировать всю программу.

  

---

  

## JMM: Happens-before: теория

  

**Happens-before** определяется как отношение между двумя действиями:

  
1. Пусть есть поток `T1` и поток `T2` (необязательно отличающийся от потока `T1`) и действия `x` и `y`, выполняемые в потоках `T1` и `T2` соответственно
2. Если `x` happens-before `y`, то во время выполнения `y` треду `T2` будут видны все изменения, выполняемые в `x` тредом `T1`
  

JLS [§17.4.5. Happens-before Order](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4.5):

  

> Two actions can be ordered by a *happens-before* relationship. If one action *happens-before* another, then the first is visible to and ordered before the second.

*Happens-before* — это еще один способ, с помощью которого мы добьемся sequential consistency. Смотрите:

  
1. Если мы свяжем conflicting доступ к shared переменной с помощью happens-before, то избавимся от data race
2. Если мы избавимся от data race, то получим sequential consistency
3. Если мы получим sequential consistency, то наша программа всегда будет выдавать консистентный с порядком в программе результат
  

Давайте сразу проясним один момент: нет, happens-before не означает, что инструкции под капотом будут действительно выполняться в таком порядке. Если переупорядочивание инструкций все равно приводит к консистентному результату, то такое переупорядочивание инструкций не запрещено. JLS:

  

> It should be noted that the presence of a happens-before relationship between two actions does not necessarily imply that they have to take place in that order in an implementation. If the reordering produces results consistent with a legal execution, it is not illegal.

Далее мы рассмотрим все действия, для которых JMM гарантирует отношение happens-before.

  

### \[Happens-Before\] Same thread actions

  

Если действие `x` идет перед `y` в коде программы и эти действия происходят в одном и том же треде, то `x` *happens-before* `y`:

  

> If `x` and `y` are actions of the same thread and `x` comes before `y` in program order, then `hb(x, y)`.

Это формальное определение as-if-serial семантики, которую я уже упоминал в начале статьи: если действие `A` идет перед действием `B` в порядке программы (program order), то `B` гарантированно увидит все изменения, которые должны быть сделаны в `A`.

  

Еще раз закрепим: happens-before не означает, что инструкции будут действительно выполняться в таком порядке под капотом. Например, давайте посмотрим на первый тред из Dekker:

  

| Thread 0 |
| --- |
| x = 1 |
| r1 = y |

  

Для этого треда гарантируется, что `write(x, 1)` happens-before `read(y)`. Однако эти действия никак не связаны: запись в `x` не влияет на чтение `y`. Другими словами, на чтении `y` нам не нужно видеть изменений, сделанных при записи в `x`. Поэтому даже если инструкции будут переупорядочены, то happens-before между этими действиями не будет нарушено.

  

Сравните:

  

| Thread 0' |
| --- |
| x = 1 |
| y = x + 1 |

  

В такой программе действия связаны — на записи в `y` нам необходимо наблюдать запись в `x`. Именно в данном случае благодаря happens-before мы увидим результат записи в `x`.

  

### \[Happens-Before\] Monitor lock

  

Освобождение монитора *happens-before* каждый последующий захват того же самого монитора.

  

> An unlock action on monitor `m` *happens-before* all subsequent lock actions on `m`

  

### \[Happens-Before\] Volatile

  

Запись в `volatile` переменную *happens-before* каждое последующее чтение той же самой переменной.

  

> A write to a volatile variable `v` happens-before all subsequent reads of `v` by any thread

  

### \[Happens-Before\] Final thread action

  

Финальное действие в треде T1 *happens-before* любое действие в треде T2, которое обнаруживает, что тред T1 завершен.

  

> The final action in a thread `T1` *happens-before* any action in another thread `T2` that detects that `T1` has terminated.

Это приводит нас к таким happens-before:

  
- Финальное действие в `T1` *happens-before* завершение вызова `T1.join()` в `T2`
- Финальное действие в `T1` *happens-before* завершение вызова `T1.isAlive()` в `T2` (если вызов возвращает `false`)
  

### \[Happens-before\] Thread start action

  

Действие запуска треда (`Thread.start()`) *happens-before* первое действие в этом треде.

  

> An action that starts a thread *happens-before* the first action in the thread it starts.

  

### \[Happens-before\] Thread interrupt action

  

Если тред `T1` прерывает тред `T2`, то интеррапт *happens-before* обнаружение интеррапта. Обнаружить интеррапт можно или по исключению `InterruptedException`, или с помощью вызова `Thread.interrupted` / `Thread.isInterrupted`.

  

> If thread `T1` interrupts thread `T2`, the interrupt by `T1` *happens-before* any point where any other thread (including `T2`) determines that `T2` has been interrupted (by having an `InterruptedException` thrown or by invoking `Thread.interrupted` or `Thread.isInterrupted`).

  

### \[Happens-Before\] Default initialization

  

Дефолтная инициализация (`0`, `false` или `null`) при создании переменной *happens-before* первое действие в каждом треде.

  

> The write of the default value (`zero`, `false`, or `null`) to each variable *happens-before* the first action in every thread.

  

### Happens-before transitivity

  

Важно отметить, что отношение *happens-before* является *транзитивным*. То есть, если `hb(x,y)` и `hb(y,z)`, *то* `hb(x,z)`.

  

Это приводит нас к одному очень важному и интересному наблюдению. Мы знаем, что два последовательных действия в одном и том же треде связаны с помощью *happens-before* (same thread actions). Тогда если действие `A` в одном треде связано отношением *happens-before* с действием `B` в другом треде, то благодаря транзитивности второму треду *во время и после выполнения действия `B`* будут видны все изменения, сделанные первым тредом *до и во время выполнения действия `A`*.

  

Еще раз: если есть последовательные действия `[A1, A2]` в первом треде, последовательные действия `[B1, B2]` во втором треде, и `hb(A2, B1)`, *то* `hb(A1, B1)`, `hb(A1, B2)` и `hb(A2, B2)`, потому что:

  
1. Для последовательных действий в треде гарантируется happens-before: `hb(A1, A2)`, `hb(B1, B2)`
2. happens-before транзитивен: если `hb(A1, A2)` (same thread), `hb(A2, B1)` (hb), `hb(B1, B2)` (same thread), *то* `hb(A1, B1)`, `hb(A1, B2)` и `hb(A2, B2)`
  

Вот как мы можем применить это знание:

  
- Не только освобождение монитора, но и все действия, идущие в порядке программы до освобождения, будут видны другому треду после захвата этого же монитора
- Не только запись в volatile поле, но и все действия, идущие в порядке программы до записи, будут видны другому треду после чтения этого же поля
- Не только финальное действие, но и все предыдущие действия треда T1 будут видны другому треду после завершения `T1.join()`
- Не только первое действие в треде, но и все последующие действия увидят дефолтную инициализацию переменной
- … не будем продолжать — идея понятна
  

Давайте с учетом этой информации запишем более полное определение *happens-before*:

  
1. Пусть есть поток `T1` и поток `T2` (необязательно отличающийся от потока `T1`) и действия `x` и `y`, выполняющиеся в потоках `T1` и `T2` соответственно
2. Если `x` happens-before `y`, то треду `T2` во время выполнения `y` и всех действий, идущих в порядке программы позже, будут видны все изменения, выполняемые тредом `T1` в `x` и всех действиях, идущих в порядке программе ранее
  

---

  

## JMM: Happens-before: практика

  

Мы уже на полпути к написанию корректных многопоточных программ — теперь осталось только применить полученные значения на практике. За основу для дальнейших примеров возьмем следующую нерабочую программу:

  
```java
public class MemoryReorderingExample {
   private int x;
   private boolean initialized = false;

   public void writer() {
      x = 5; /* W1 */
      initialized = true; /* W2 */
   }

   public void reader() {
      boolean r1 = initialized; /* R1 */
      if (r1) {
         int r2 = x; /* R2, may read default value (0) */
      }
   }
}
```
  

Можно подумать, что если мы прочитали значение `true` на `R1`, то прочитаем и значение `5` на `R2`, так как в порядке программы запись в `x` идет перед записью в `initialized`. Но на самом деле мы можем наблюдать значение по умолчанию (`0`) при чтении `x` по следующим причинам:

  
1. *Instructions reordering (1/2)* — записи W1 и W2 были переставлены местами
2. *Instructions reordering (2/2)* — чтения R1 и R2 были переставлены местами
3. *Visibility* — запись в `x` не пропагирована другим ядрам на момент чтения
  

С точки зрения программы мы говорим, что произошли `StoreStore` или `LoadLoad` переупорядочивания. Это ожидаемо, ведь мы имеем две гонки: при доступе к `x` и `initialized`. А соответственно, нам не гарантируются только sequentially consistent выполнения, ведь такие переупорядочивания валидны с точки зрения JMM при наличии гонок.

  

Давайте лично убедимся в том, что такие переупорядочивания возможны, написав jcstress тест:

  
```java
@JCStressTest
@Description("Triggers memory reordering")
@Outcome(id = "-1", expect = Expect.ACCEPTABLE, desc = "Not initialized yet")
@Outcome(id = "5", expect = Expect.ACCEPTABLE, desc = "Returned correct value")
@Outcome(id = "0", expect = Expect.ACCEPTABLE_INTERESTING, desc = "Initialized but returned default value")
public class JmmReorderingPlainTest {

   @Actor
   public final void actor1(DataHolder dataHolder) {
      dataHolder.writer();
   }

   @Actor
   public final void actor2(DataHolder dataHolder, I_Result r) {
      r.r1 = dataHolder.reader();
   }

   @State
   public static class DataHolder {
      private int x;
      private boolean initialized = false;

      public void writer() {
         x = 5;
         initialized = true;
      }

      public int reader() {
         if (initialized) {
            return x;
         }
         return -1; // return mock value if not initialized
      }
   }
}
```
  

Запускаем тест на Intel Core i7-11700 (x86), Windows 10 x64, OpenJDK 17 и получаем следующие результаты:

  
```
Results across all configurations:

RESULT        SAMPLES     FREQ       EXPECT  DESCRIPTION
    -1  5,004,050,680   38,73%   Acceptable  Not initialized yet
     0        168,651   <0,01%  Interesting  Initialized but returned default value
     5  7,916,756,029   61,27%   Acceptable  Returned correct value
```
  

Как видите, в <0,01% случаев мы получили неконсистентный с порядком в программе результат.

  

Далее мы доведем эту программу до полной корректности, используя happens-before. Нашей целью будет связать доступ к этим переменным с помощью happens-before, чтобы избавиться от гонок и добиться только sequentially consistent выполнений.

  

### Monitor lock

  

[Monitor lock](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html) не только предоставляет happens-before между освобождением и взятием лока, но также является и [мьютексом](https://ru.wikipedia.org/wiki/%D0%9C%D1%8C%D1%8E%D1%82%D0%B5%D0%BA%D1%81), который позволяет обеспечить эксклюзивный доступ к критической секции. Каждый объект в Java содержит внутри себя такой лок, но его нельзя использовать напрямую — чтобы воспользоваться им, необходимо применить keyword `synchronized` (отсюда и альтернативное название *intrinsic lock*).

  

Вот как мы можем исправить приведенную выше программу с помощью монитора:

  
```java
public class SynchronizedHappensBefore {
   private final Object lock = new Object();

   private int x;
   private boolean initialized = false;

   public void writer() {
      synchronized (lock) {
         x = 5; /* W1 */
         initialized = true; /* W2 */
      } /* RELEASE */
   }

   public synchronized void reader() {
      synchronized (lock) { /* ACQUIRE */
         boolean r1 = initialized; /* R1 */
         if (r1) {
            int r2 = x; /* R2, guaranteed to see 5 */
         }
      }
   }
}
```
  

В данном примере мы используем монитор объекта `lock`, свойство happens-before которого гарантирует следующее: после получения монитора `reader` увидит все изменения, которые идут в порядке программы перед освобождением монитора в треде `writer`. Следите внимательно: если `hb(W1, W2)` (same thread), `hb(W2, RELEASE)` (same thread), `hb(RELEASE, ACQUIRE)` (monitor lock), `hb(ACQUIRE, R1)` (same thread), `hb(R1, R2)` (same thread), *то `hb(W2, R1)` и `hb(W1, R2)` (transitivity)*.

  

Таким образом, мы имеем happens-before как при доступе к `x` (`hb(W1, R2)`), так и при доступе к `initialized` (`hb(W2, R1)`), а значит наша программа больше не имеет гонок. Тогда любой результат выполнения программы будет объясним одним из следующих SC порядков:

  
```
monitorEnter() -> write(x, 5) -> write(initialized, true) -> monitorExit() -> monitorEnter() -> read(initialized):true -> read(x):5 -> monitorExit() // (initialized, x) = (true, 5)
monitorEnter() -> read(initialized):false -> monitorExit() -> monitorEnter() -> write(x, 5) -> write(initialized, true) -> monitorExit() // (initialized, x) = (false, _)
```
  

---

  

### Volatile

  

*Volatile* предоставляет happens-before гарантию между записью и чтением из `volatile` переменной. Семантика volatile отличается от монитора только тем, что не предоставляет эксклюзивный доступ к критической секции.

  

Вот так с помощью `volatile` мы исправляем ту же самую программу:

  
```java
public class VolatileHappensBefore {

   private int x;
   private volatile boolean initialized;

   public void writer() {
      x = 5; /* W1 */
      initialized = true; /* W2 */
   }

   public void reader() {
      boolean r1 = initialized; /* R1 */
      if (r1) {
         int r2 = x; /* R2, guaranteed to see 5 */
      }
   }
}
```
  

В данном примере мы синхронизируемся на `volatile` поле `initialized`, свойство happens-before которого гарантирует следующее: после обнаружения записи `reader` увидит все изменения, которые идут в порядке программе перед записью в переменную в треде `writer`. Следите внимательно: если `hb(W1, W2)` (same thread), `hb(W2, R1)` (volatile), `hb(R1, R2)` (same thread), *то `hb(W1, R2)` (transitivity)*.

  

Таким образом, аналогично мы имеем happens-before как при доступе к `x` (`hb(W1, R2)`), так и при доступе к `initialized` (`hb(W2, R1)`). Тогда любой результат выполнения программы будет объясним одним из следующих SC порядков:

  
```
write(x, 5) -> write(initialized, true) -> read(initialized):true -> read(x):5 // (initialized, x) = (true, 5)
write(x, 5) -> read(initialized):false -> write(initialized, true) // (initialized, x) = (false, _)
read(initialized):false -> write(x, 5) -> write(initialized, true) // (initialized, x) = (false, _)
```
  

Здесь по сравнению с предыдущим примером возможных SC порядков побольше, так как нет эксклюзивного захвата лока, но итоговый сет результатов все равно остается таким же.

  

---

  

## Safe Publication/RA semantics

  

Наверняка вы заметили, что в обоих примерах прослеживается некая идиома: делаем все необходимые изменения до некоторого синхронизирующего треды действия `A` (запись в `volatile` переменную, освобождение лока), а читаем эти изменения после другого синхронизирующего действия `B` (чтение `volatile` переменной, взятие лока) — это называется *[acquire/release семантика](https://preshing.com/20120913/acquire-and-release-semantics/)* или *Safe Publication* (безопасная публикация). Наверняка, многие из вас слышали последнее понятие.

  

Можно сказать, что release действие паблишит все изменения другим тредам, а acquire действие принимает эти изменения. Если говорить более формально, то release семантика запрещает переупорядочивание release действия с записями, идущими в порядке программе ранее (StoreStore), а acquire семантика запрещает переупорядочивание acquire действия с чтениями, идущими в порядке программы позже (LoadLoad). Конечно, это всего лишь высокоуровневая интерпретация свойств happens-before и таких терминов вы не найдете в спеке, но для простоты понимания можно вполне мыслить в рамках "безопасной публикации".

  

| Thread 1 | Thread 2 |
| --- | --- |
| ...writes... |  |
| release | acquire |
|  | ...reads... |

  

Именно благодаря тому, что happens-before порядок не требует строгого порядка для обычных записей/чтений, все действия до release или после acquire могут быть переупорядочены под капотом как угодно. Но главное то, что они не будут переупорядочены с самим release/acquire действием. Например, если мы имеем действия `[W1, W2, W3, RELEASE]`, то `[W1, W2, W3]` могут быть переупорядочены под капотом как угодно, но они всегда будут выполнены до `RELEASE` действия.

  

Безопасная публикация распространяется и на объекты — взгляните на следующий пример:

  
```java
public class JmmReorderingObjectExample {
    private Foo instance;

    private static class Foo {
        private int x;

        Foo() {
            this.x = 5; /* W1 */
        }
    }

    public void writer() {
        instance = new Foo(); /* W2, non-safe publish */
    }

    public void reader() {
        Foo r1 = instance; /* R1 */
        if (r1 != null) {
           int r2 = r1.x; /* R2: may be default value (0) */
        }
    }
}
```
  

В данной программе мы можем наблюдать дефолтное значение на чтении `R2` из-за StoreStore переупорядочивания (запись ссылки `W2` произошла перед записью в конструкторе `W1`) или LoadLoad переупорядочивания (чтение `R2` случилось перед чтением `R1`).

  

Но как только мы пометим переменную `instance` как `volatile` или будем читать/писать под монитором, то такие переупорядочивания будут невозможны благодаря транзитивности happens-before:

  
```
write(x, 5) --hb-> write(volatile instance, ref) --hb-> read(volatile instance):ref -> read(x): 5
monitorEnter() --hb-> write(x, 5) --hb-> write(instance, ref) --hb-> monitorExit() --hb-> monitorEnter() --hb-> read(instance):ref --hb-> read(x): 5 --hb-> monitorExit()
```
  

Смотрите jcstress тесты, наглядно демонстрирующие безопасную публикацию объектов — [https://github.com/blinky-z/JmmArticleHabr/tree/main/jcstress/tests/object](https://github.com/blinky-z/JmmArticleHabr/tree/main/jcstress/tests/object).

  

Как видите, пользоваться happens-before достаточно просто. Анализируйте свою программу: в большинстве случаев применима именно безопасная публикация, которая гарантирует вам видимость всех сделанных изменений. И только в редких случаях вам придется связать все действия в Synchronization Order как в случае с Dekker локом. В любом случае, оба способа гарантируют вам отсутствие гонок и sequentially consistent результаты.

  

---

  

## Happens-before is about actions

  

После того как мы рассмотрели работу happens-before на практике, я бы хотел подробнее остановиться на одном важном моменте. Может кому-то все далее написанное покажется очевидным, но разобраться не помешает.

  

Вы уже достаточно много раз увидели слово "действие" (action) пока читали раздел про happens-before, но скорее всего смысл его не до конца очевиден. Так вот, под "действием" спека имеет в виду результат выполнения выражения в рантайме, а не само статическое выражение в коде (statement). Например, `boolean r1 = initialized;` это всего лишь выражение в программе, но оно может иметь результат: чтение `true` или `false`. Это и есть действие, которое имеет определенный результат во время выполнения. Действия следует рассматривать свободно и абстрактно от различных переупорядочиваний под капотом: просто смотрите на порядок выражений в коде программы, но думайте о том, какой результат может иметь выражение.

  

Теперь следует сказать, что happens-before (и все остальные порядки в JMM) — это именно о *действиях* (actions), которые имеют какой-то результат, а не об абстрактных выражениях в программе.

  

Давайте еще раз взглянем на определение happens-before для треда в изоляции:

  

> If `x` and `y` are *actions* of the same thread and `x` comes before `y` in program order, then `hb(x, y)`.

Как видите, здесь говорится именно о *действиях* (actions): два любых действия в одном и том же треде связаны happens-before, если их соответствующие выражения идут последовательно в порядке программы. Приведу отдельный пример: happens-before существует не между выражениями `x = i` и `int r1 = x`, а между действиями записи `write(x, 5)` и чтением `read(x)`, которое гарантированно обнаружит `5` благодаря установке hb между ними (допустим, что при выполнении `i` имеет значение `5`). Здесь все просто.

  

А вот в случае `volatile` все немного интереснее. Вот определение hb для volatile:

  

> A write to a volatile variable `v` happens-before all *subsequent* reads of `v` by any thread

Обратите внимание на слово *"subsequent"* — это очень важная деталь, которая различает действия и выражения. Определение говорит, что чтение должно *обнаружить* эту запись, чтобы между записью и чтением `volatile` переменной было установлено happens-before отношение. То есть, здесь подразумевается некоторый результат выполнения — это и есть "действие".

  

Именно поэтому даже в случае использования `volatile` *все еще необходимо* делать условие `if (initialized)`, ведь только при обнаружении записи устанавливается отношение `hb` между двумя тредами. То есть, определение happens-before для `volatile` не говорит, что в рантайме чтение всегда увидит `true`, ведь это зависит от шедулера JVM/ОСи и времени пропагации записи другим тредам (eventual visibility). Определение говорит, что *если* запись была обнаружена, то будет установлено отношение hb между тредами. Аналогично следует рассуждать и о мониторе: необходимо обнаружить освобождение монитора, чтобы было установлено hb между тредами.

  

Да, все это звучит банально, но я лишь хотел предостеречеть вас от написания такой программы:

  
```java
public class VolatileNotAlwaysHappensBefore {
   private int x;
   private volatile boolean initialized;

   public void writer() {
      x = 5; /* W1 */
      initialized = true; /* W2 */
   }

   public void reader() {
      boolean r1 = initialized; /* R1 */
      int r2 = x; /* R2, ??? 5 or 0 */
   }
}
```
  

Где вы могли бы неверно рассуждать, что на `R2` всегда прочитаем `5`, ведь типа запись в volatile переменную happens-before чтение из нее, а че бы нет? Но дело в том, что вот так есть hb между тредами:

  
```
write(x, 5) --hb-> write(initialized, true) --hb-> read(initialized):true --hb-> read(x):5
```
  

А вот так нет:

  
```
write(x, 5) --hb-> write(initialized, true) ... read(initialized):false --hb-> read(x):?
```
  

Подводя итог, этим разделом я хотел сказать следующее:

  
1. Happens-before — это о действиях, которые имеют какой-то результат, а не об абстрактных выражениях в программе. Следите внимательно за тем, при каких условиях устанавливается hb, и анализируйте свой случай
2. Для synchronization actions отношение happens-before устанавливается только при обнаружении определенного эффекта памяти. Например, для `volatile` переменной hb устанавливается только после обнаружения записи, а для монитора после обнаружения его освобождения
3. Для действий в одном и том же треде happens-before устанавливается всегда
  

---

  

## Cache Coherence

  

В самом начале статьи я уже затрагивал тему Cache Coherence, а теперь разберемся в ней подробнее.

  

Перед тем как идти дальше, рассмотрим устройство кэша на базовом уровне:

  
1. Процессор никогда не работает с памятью напрямую — все операции чтения и записи проходят через кэш. Когда процессор хочет загрузить значение из памяти, то он обращается в кэш. Если значения там нет, то кэш сам ответственнен за выгрузку значения из памяти с последующим сохранением в кэше. Когда процессор хочет записать значение в память, то он записывает значение в кэш, который в свою очередь ответственен за сброс значения в память
2. Кэш состоит из множества "линий" (cache line) фиксированного размера, в которые кладутся значения из памяти. Размер линий варьируется от 16 до 256 байт в зависимости от архитектуры процессора. Кэш сам знает, как мапить адрес линии кэша в адрес памяти
3. Кэш имеет фиксированный размер, поэтому может хранить ограниченное количество записей. Например, если размер кэша 64 KB, а размер линии кэша 64 байт, то всего кэш может содержать 1024 линии. Поэтому, если при выгрузке нового значения места в кэше не хватает, то из кэша вымещается одно из значений
4. Большинство современных архитектур процессоров имеют [несколько уровней кэша](https://en.wikipedia.org/wiki/Cache_hierarchy): обычно это L1, L2, и L3. Верхние уровни кэша (L1, L2) являются локальными — каждое ядро процессора имеет собственный, отдельный от других ядер кэш. Кэш на самом нижнем уровне (L3) является общим и шарится между всеми ядрами  
	1. Доступ к каждому последующему уровню кэша стоит дороже, чем к предыдущему. Например, доступ к L1 может стоить 3 цикла, L2 — 12 циклов, а к L3 — 38 циклов
	2. Каждый последующий кэш имеет больший размер, чем предыдущий. Например, L1 может иметь размер 80 KB, L2 — 1.25 MB, а L3 — 24 MB
  

Из-за того, что ядра имеют собственный локальный кэш, возникает потенциальная проблема чтения неактуальных значений. Например, пусть два ядра прочитали одно и то же значение из памяти и сохранили в свой локальный кэш. Затем первое ядро записывает новое значение в свой локальный кэш, но другое ядро не видит этого изменения и продолжает читать устаревшее значение. Как итог, данные среди локальных кэшей не консистентны. Если бы в процессоре существовал только общий кэш, то проблемы чтения неактуальных значений просто не существовало бы: так как все записи и чтения проходят через кэш, а не идут напрямую в память, то общий кэш по сути был бы master копией памяти, где всегда лежали бы актуальные значения. Но это сильно ударило бы по производительности процессора, так как кэш может обрабатывать только один цикл единовременно, а значит ядра простаивали бы в очереди. Более того, локальный кэш распаян физически ближе к ядру, поэтому доступ к нему стоит дешевле. Именно поэтому и необходим локальный кэш, чтобы каждое ядро могло эффективно работать с кэшем независимо от других ядер.

  
![Структура кэша](https://habrastorage.org/r/w1560/webt/at/pw/m2/atpwm2vnpon_q-28tuwueoltb7s.jpeg)  

На самом деле, процессоры умеют поддерживать консистентность данных среди локальных кэшей так, что любое из ядер всегда читает актуальное значение одного и того же адреса памяти.

  

**[Cache Coherence](https://en.wikipedia.org/wiki/Cache_coherence)** (когерентность кэша) — это механизм процессора, гарантирующий, что любое ядро всегда читает самое актуальное значение из кэша. Данным механизмом обладают многие современные архитектуры процессоров в той или иной имплементации. Самый популярный из протоколов — это [MESI](https://en.wikipedia.org/wiki/MESI_protocol) протокол и его производные. Например, Intel использует [MESIF](https://en.wikipedia.org/wiki/MESIF_protocol), а AMD — [MOESI](https://en.wikipedia.org/wiki/MOESI_protocol) протокол.

  

В MESI протоколе линия кэша может находиться в одном из следующих состояний:

  
1. ***I** nvalid* — линия кэша устарела (содержит неактуальные значения), поэтому из нее нельзя читать
2. ***S** hared* — линия кэша актуальна и эквивалентна памяти. Процессор может только читать из такой линии кэша, но не писать в нее. Если несколько ядер читают один и тот же адрес памяти, то эта линия кэша будет реплицирована сразу в несколько локальных кэшей, отсюда и название "shared"
3. ***E** xclusive* — линия кэша актуальна и эквивалентна памяти. Однако как только одно из ядер процессора переводит линию кэша в это состояние, никакое другое ядро не может держать эту линию кэша у себя, отсюда и название "exclusive". Когда значение из памяти только первые загружается в кэш, то линия кэша устанавливается именно в это состояние. Если одно из ядер процессора хочет перевести линию кэша из *shared* в *exclusive* состояние, то все остальные ядра должны пометить свою копию как *invalid*
4. ***M** odified* — линия кэша была изменена (dirty), то есть ядро записало в нее новое значение. Именно в это состояние переходит exclusive линия кэша после записи в нее. Аналогично, только одно из ядер процессора может держать линию кэша в Modified состоянии. Если линия вымещается из кэша, то кэш ответственен за то, чтобы записать новое значение в память перед выгрузкой
  

Когда одно из ядер процессора хочет изменить линию кэша, то оно должно установить exclusive доступ к ней. Для этого ядро посылает всем остальным ядрам сообщение о том, что указанную линию кэша необходимо пометить как invalid в их локальном кэше. Только после того, как ядра обработают запрос, пометив свою копию как invalid, ядро сможет записать новое значение вместе с этим помечая линию кэша как modified. Таким образом, при записи только одно ядро может удерживать значение в локальном кэше, а значит неконсистентность данных просто невозможна.

  

Когда любое ядро хочет прочитать какой-нибудь адрес в памяти, то алгоритм действий выглядит так:

  
1. Ядро обращается в L1 кэш и проверяет, присутствует ли там искомое значение. Если линия кэша присутствует и находится в состоянии Shared, Exclusive или Modified, то происходит ее чтение. Если значение в локальном кэше не обнаружено (или линия кэша находится в состоянии Invalid), то говорится, что произошел (local) "cache miss"
2. По специальной общей шине всем остальным ядрам передается запрос на чтение значения. Все остальные ядра видят этот запрос, и если одно из ядер содержит искомое значение в состоянии Shared, Exclusive или Modified, то оно отдает актуальное значение в ответ.  
	- Если линия кэша была установлена в Modified состояние, то перед тем как отдать значение, измененное значение сбрасывается в память, а затем линия кэша переводится в Shared состояние
3. Если значение не обнаружено ни в одном из локальных кэшей, то происходит чтение из памяти
4. Вне зависимости от того, где мы нашли значение, читающее ядро сохраняет данные в свой локальный кэш, помечая линию кэша как shared
  

Это очень упрощенное описание работы кэша — я опустил многие детали, но надеюсь, что примерная картина вам понятна. Скажу сразу, что я не претендую на полную корректность вышенаписанного: где-то я мог и соврать, ибо не являюсь специалистом в такой низкоуровневой теме как процессоры. Более того, многие моменты могут отличаться в зависимости от микроархитектуры процессора и используемого Cache Coherence протокола. В конце статьи я приведу ссылки на другие полезные источники, где вы сможете узнать подробнее о работе кэша.

  

Таким образом, как только значение попадает в локальный кэш, оно *сразу же становится видно другим ядрам*.

  

Теперь наверняка у вас возник закономерный вопрос: так что же, значит visibility проблемы на уровне процессора не существует? На самом деле, не все так просто.

  

### Invalidation Queue

  

Когда ядро получает запрос на инвалидацию записи в кэше, он может быть обработан не сразу, а поставиться в очередь *Invalidation Queue* (IQ). Эта оптимизация необходима по следующим причинам: во-первых, ядро может быть занято другой работой, и во-вторых, мы хотим, чтобы при большом количестве запросов ядро не заблокировалось на долгое время в их обработке, а обработало все постепенно. Таким образом, можно сказать, что invalidate запросы являются асинхронными

  

Проблема в том, что мы рискуем не прочитать самое актуальное значение просто потому, что запрос в invalidation queue еще не был обработан, а в кэше лежало еще не инвалидированное, но уже устаревшее значение.

  

Например:

  

| CORE 0 | CORE 1 |
| --- | --- |
| Cached (shared): x(5) | Cached (shared): x(5) |
| send invalidate request |  |
|  | accept invalidate request, put in IQ and respond with acknowledgement |
| Cached (exclusive): x(5) | Cached (shared): x(5) |
| x = 10 |  |
|  | r = x // 5 |
|  | handle invalidate request // too late! |

  

Как видите, мы прочитали устаревшее значение, хотя запрос на invalidate уже пришел.

  

### Store Buffer

  

В некоторых микро-архитектурах (как x86) каждое ядро имеет локальный FIFO *Store Buffer* (SB, write buffer), который является прослойкой между CPU и кэшем. В этот буфер ядро кладет все записи, которые будут ожидать там сброса в локальный кэш до тех пор, пока все остальные ядра не инвалидируют эту запись в своем кэше и не пришлют acknowledgement. Эта оптимизация требуется для того, чтобы не задерживать работу пишущего ядра, пока остальные ядра обрабатывают запрос на инвалидацию. При чтении ядро сперва смотрит в свой SB перед тем, как идти в локальный кэш, чтобы избежать чтения неактуальных значений и таким образом поддержать as-if-serial гарантию внутри одного ядра

  

Проблема в том, что другие ядра не увидят новой записи, пока пишущее ядро не сбросит запись из SB в локальный кэш, так как SB — это часть ядра, но не кэша. Другими словами, Cache Coherence механизм не распространяется на Store Buffer. Соответственно, некоторый промежуток времени пишущее ядро будет оперировать актуальным значением, но все остальные — устаревшим.

  

Например:

  

| CORE 0 | CORE 1 |
| --- | --- |
| Cached (exclusive): x(0) | Cached: none |
| x = 5 // put in SB |  |
| r2 = x // 5, read from SB |  |
|  | r1 = x // 0 |
| flushed from sb to cache |  |
| r3 = x // 5, read from local cache |  |

  

Как видите, CORE 0 произвело запись в `x`, а затем CORE 1 пытается прочитать эту переменную. Однако CORE 1 не найдет актуального значения ни в памяти, ни в кэше CORE 0, так как эта запись все еще лежит в Store Buffer. Соответственно, CORE 1 увидит `0` на чтении `r1`, хотя CORE 0 оперирует актуальным значением на `r2`, чем нарушается консистентность данных.

  

Итак, ядра действительно всегда видят актуальное значение, но только *кроме короткого временного окна после записи*. Другими словами, нам гарантируется *eventual visibility* изменений.

  

В заключение приведу полное устройство кэша:

  
![Полная структура кэша](https://habrastorage.org/r/w1560/webt/qc/z4/nm/qcz4nmuu6fp2z-f2wv623dfn_fc.jpeg)  

---

  

## Eventual Visibility

  

Можно наивно предположить, что благодаря Cache Coherence нам гарантируется eventual visibility и на уровне Java для обычных записей и чтений, то есть не связанных happens-before. Однако, это не правда, так как мы работаем на уровне языка, а не процессора. Компилятор может оптимизировать код так, что запись никогда не станет видна другому треду. Яркий пример — это такой busy wait, где в бесконечном цикле проверяется значение shared переменной.

  

JCStress уже имеет готовый тест для этого случая — [BasicJMM\_04\_Progress#PlainSpin](https://github.com/openjdk/jcstress/blob/83365e01e6606f8368a74b3e5503361f51074cde/jcstress-samples/src/main/java/org/openjdk/jcstress/samples/jmm/basic/BasicJMM_04_Progress.java#L41):

  
```java
@JCStressTest(Mode.Termination)
@Outcome(id = "TERMINATED", expect = ACCEPTABLE,             desc = "Gracefully finished")
@Outcome(id = "STALE",      expect = ACCEPTABLE_INTERESTING, desc = "Test is stuck")
@State
public static class PlainSpin {
   boolean ready;

   @Actor
   public void actor1() {
      while (!ready); // spin
   }

   @Signal
   public void signal() {
      ready = true;
   }
}
```
  

Смотрим на результаты запуска теста:

  
```
RESULT  SAMPLES     FREQ       EXPECT  DESCRIPTION
       STALE        4   50.00%  Interesting  Test is stuck
  TERMINATED        4   50.00%   Acceptable  Gracefully finished
```
  

Как видите, в половине случаев тред завис навсегда. Это произошло по той причине, что компилятор оптимизировал цикл `while (!ready)` в `while(true)`. Компилятор свободен это делать, так как переменная не изменяется ни до, ни внутри цикла, а также не связана отношением happens-before с действиями в других тредах.

  

Исправить этот пример можно пометив переменную как [volatile](https://github.com/openjdk/jcstress/blob/83365e01e6606f8368a74b3e5503361f51074cde/jcstress-samples/src/main/java/org/openjdk/jcstress/samples/jmm/basic/BasicJMM_04_Progress.java#L74) или работая с переменной под [монитором](https://github.com/openjdk/jcstress/blob/83365e01e6606f8368a74b3e5503361f51074cde/jcstress-samples/src/main/java/org/openjdk/jcstress/samples/jmm/basic/BasicJMM_04_Progress.java#L147) — только в этом случае нам гарантируется eventual visibility изменений.

  

Таким образом, пока мы работаем с обычными записями и чтениями, не связанными отношением happens-before, нам не гарантируется видимость изменений, сделанных из других тредов.

  

---

  

## Memory Barriers

  

Процессор может переупорядочивать выполняемые им инструкции, даже если на уровне компилятора мы обеспечили необходимый порядок. Хотя процессор делает только такие переупорядочивания, которые не меняют итогового результата, но это гарантируется только для единственного ядра в изоляции, поэтому переупорядочивание может повлиять на другие ядра. Более того, все еще существует проблема видимости изменений, которую мы обсудили выше. Именно поэтому JMM ответственна и за синхронизацию на уровне процессора.

  

Для решения этих проблем Java использует готовые низкоуровневые механизмы синхронизации под названием "memory barrier", предоставляемые самим процессором. Задача барьеров памяти — запретить (memory) переупорядочивания, которые обычно разрешены моделью памяти процессора. Таким образом, точно так же как мы используем примитивы синхронизации `volatile` / `synchronized` в высокоуровневом коде, сама Java под капотом тоже использует похожие низкоуровневые примитивы синхронизации.

  

**[Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)** (memory fence, барьер памяти) — это тип процессорной инструкции, которая заставляет процессор гарантировать memory ordering для инструкций, работающих с памятью.

  

Всего существует 4 типа барьеров памяти — они напрямую матчатся в возможные memory reordering и запрещают каждый из них:

  
1. *LoadLoad*  
	- дает гарантию, что все load операции до барьера произойдут перед load операциями после барьера
2. *LoadStore*  
	- дает гарантию, что все load операции до барьера произойдут перед store операциями после барьера
3. *StoreStore*  
	- дает гарантию, что все store операции до барьера произойдут перед store операциями после барьера. Таким образом, все store операции до барьера будут тоже видны, если станет видна любая store операция после барьера
4. *StoreLoad*  
	- дает гарантию, что все store операции до барьера произойдут перед load операциями после барьера. Таким образом, все store операции до барьера станут видны другим ядрам перед тем, как произойдет любая load операция после барьера
  

То, как имплементированы барьеры — это дело процессора. К примеру, они могут запрещать переупорядочивание инструкций и ожидать полной обработки Store Buffer/Invalidation Queue, но мы не знаем точной имплементации. На самом деле, знание таких деталей и не нужно — мы просто мыслим в терминах Memory Ordering и тех гарантий порядка, которые дают нам барьеры.

  

Соответствующие процессорные инструкции или отображаются 1-в-1 в эти типы, или же объединяют в себе сразу несколько типов барьеров. Все процессоры имеют как минимум одну full memory barrier инструкцию, которая объединяет в себя сразу все типы барьеров, запрещая memory reordering как load, так и store инструкций вокруг барьера. Например, на x86 мы имеем [mfence](https://www.felixcloutier.com/x86/mfence) и [lock prefix](https://www.felixcloutier.com/x86/lock), которые являются full memory barrier. Однако процессоры могут предоставлять и более дешевые, гранулярные барьеры памяти.

  

Обычно Load- и Store- барьеры используются в паре: Store барьер гарантирует, что записи будут видны другому ядру, а Load барьер гарантирует, что чтения будут выполнены в необходимом порядке.

  

Например, вот как мы можем исправить уже знакомый нам по вступлению пример с помощью барьера:

  

| Thread 0 | Thread 1 |
| --- | --- |
| x = 1 | y = 1 |
| \[StoreLoad\] | \[StoreLoad\] |
| r1 = y | r2 = x |

  

Если мы поставим `StoreLoad` барьер после записи, то процессору запрещается переупорядочивать store инструкции до барьера с load инструкциями после барьера. В такой программе мы можем быть точно уверены, что не получим результата `(r1, r2) = (0, 0)`.

  

Давайте лично убедимся в наличии барьеров под капотом Java на примере `volatile`. В [JSR-133 Cookbook](https://gee.cs.oswego.edu/dl/jmm/cookbook.html), неофициальном гайдлайне по имплементации JMM за авторством Doug Lea, сказано:

  

> 1. Issue a `StoreStore` barrier before each volatile store.
> 2. Issue a `StoreLoad` barrier after each volatile store.
> 3. Issue `LoadLoad` and `LoadStore` barriers after each volatile load.
>   

Так, давайте здесь прервемся и проясним несколько моментов:

  
1. JSR-133 Cookbook — это не то, как имплементирована JMM на самом деле во всех JVM. JMM может не соответствовать Cookbook и наоборот. Cookbook — это всего лишь то, как *может быть* имплементирована JMM. Но для нашей статьей этого достаточно, так как там дается примерное корректное применение барьеров
2. Если вы вдруг посчитаете, что все поняли, и решите отойти от формализмов и абстракций JMM, начав использовать барьеры напрямую, могут произойти плохие вещи. Барьеры — это деталь имплементации, а не официальные гарантии JMM. Более того, барьеры сильно разнятся от микроархитектуры к микроархитектуре и вы не знаете как они работают на самом деле и как ими вообще пользоваться. Используйте только то, что дает вам JMM, а именно абстрактные synchronization order/happens-before
  

Теперь продолжаем. Пусть есть такая простая программа с использованием `volatile`:

  
```java
public class VolatileMemoryBarrierJIT {

   private static int field1;
   private volatile static int field2;

   private static void write(int i) {
      field1 = i << 1;
      /* StoreStore */
      field2 = i << 2;
      /* StoreLoad */
   }

   private static void read() {
      int r1 = field2;
      /* LoadLoad + LoadStore */
      int r2 = field1;
   }

   public static void main(String[] args) throws Exception {
      // invoke JIT
      for (int i = 0; i < 10000; i++) {
         write(i);
         read();
      }
      Thread.sleep(1000);
   }
}
```
  

Теперь возьмем дизассемблер [hsdis](https://blogs.oracle.com/javamagazine/post/java-hotspot-hsdis-disassembler) и посмотрим на сгенерированный JIT-компилятором нативный код. Запускаем дизассемблер на Intel Core i7-11700 (x86), Windows 10 x64, OpenJDK 17. Вот сгенерированный ASM код для `write()`:

  
```
[Verified Entry Point]
  # {method} {0x00000175a1400310} 'write' '(I)V' in 'jit_disassembly/VolatileMemoryBarrierJIT'
  # parm0:    rdx       = int
  #           [sp+0x40]  (sp of caller)
  0x000001758817dae3:   mov    DWORD PTR [rsi+0x70],edi     ;*putstatic field1 {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.VolatileMemoryBarrierJIT::write@3 (line 9)
  0x000001758817dae6:   shl    edx,0x2
  0x000001758817dae9:   mov    DWORD PTR [rsi+0x74],edx
  0x000001758817daec:   lock add DWORD PTR [rsp-0x40],0x0   ;*putstatic field2 {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.VolatileMemoryBarrierJIT::write@9 (line 10)
```
  

В `mov` инструкциях мы записываем значения полей `field1` / `field2`, причем в порядке программы. Теперь обратите внимание на инструкцию `lock add DWORD PTR [rsp-0x40],0x0`. Это может показаться странным, что мы добавляем `0` к значению на стеке (`rsp`), но эта инструкция выступает лишь в качестве дешевой по стоимости "заглушки". Все дело в наличии `lock` префикса, который является full memory barrier на x86, что и дает нам `StoreLoad` барьер после записи в `volatile`. JVM могла бы использовать `mfence` барьер, но на современных процессорах `lock add` с добавлением 0 на стек [является эффективнее](https://web.archive.org/web/20110620202202/https://blogs.oracle.com/dave/entry/instruction_selection_for_volatile_fences).

  

Наверняка у вас возник вопрос: где же `StoreStore` барьер? Как мы уже видели во вступлении, x86 дает достаточно сильные гарантии порядка. Из [Intel Software Developer's Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf#G13.14501):

  

> - Reads are not reordered with other reads \[запрещает LoadLoad reordering\]
> - Writes are not reordered with older reads \[запрещает LoadStore reordering\]
> - Writes to memory are not reordered with other writes \[запрещает StoreStore reordering\]
> - Reads may be reordered with older writes to different locations but not with older writes to the same location \[разрешает StoreLoad reordering\]
>   

Из этого следует, что нет необходимости использовать `LoadLoad`, `LoadStore`, и `StoreStore` барьеры на x86 микроархитектуре, а нужен только `StoreLoad` барьер. JVM достаточно умна, чтобы не использовать дорогие барьеры памяти там, где процессор уже дает необходимые гарантии, поэтому мы и не видим применения барьера в сгенерированном нативном коде.

  

Теперь посмотрим на ASM код для `read()`:

  
```
[Verified Entry Point]
  # {method} {0x00000175a14003a8} 'read' '()V' in 'jit_disassembly/VolatileMemoryBarrierJIT'
  #           [sp+0x40]  (sp of caller)
  0x000001758817de5e:   mov    edi,DWORD PTR [rsi+0x74]     ;*getstatic field2 {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.VolatileMemoryBarrierJIT::read@0 (line 14)
  0x000001758817de61:   mov    esi,DWORD PTR [rsi+0x70]     ;*getstatic field1 {reexecute=0 rethrow=0 return_oop=0}
                                                            ; - jit_disassembly.VolatileMemoryBarrierJIT::read@4 (line 15)
```
  

И снова заметим, что барьеры `LoadLoad` и `LoadStore` отсутствуют при чтении `volatile` переменной благодаря строгим гарантиям x86 микроархитектуры. Однако на более слабой микроархитектуре как ARM мы будем наблюдать барьеры в этих местах (смотрите [volatile\_jit\_asm\_arm64.txt](https://github.com/blinky-z/JmmArticleHabr/blob/main/disassembly/arm64/volatile_jit_asm_arm64.txt)).

  

---

  

## Как JMM обеспечивает консистентный memory order: подводим итоги

  

Итак, давайте резюмируем, что делает JMM на каждом из уровней, чтобы правильно синхронизированная программа не имела переупорядочиваний. Happens-before — это конечно хорошо, но это всего лишь абстракция. А вот на нижнем уровне компилятора и хардвара JMM на самом деле делает следующее:

  
1. Compiler memory ordering  
	1. Уровень компилятора байткода (`javac`)  
		- Обеспечивает такой порядок сгенерированных bytecode инструкций, который будет консистентен с порядком действий в коде
	2. Уровень компилятора машинного кода (`HotSpot JIT Compiler C1/C2`)  
		- Обеспечивает такой порядок сгенерированных машинных инструкций, который будет консистентен с порядком действий в коде
2. CPU memory ordering  
	- Расставляет барьеры памяти в нужных местах так, чтобы memory ordering машинных инструкций был консистентен с порядком действий в коде
  

Первые два уровня зависят полностью от самой Java — именна она имплементирует гарантию порядка. Уровень процессора же зависит не только от Java, но и от самого процессора, который предоставляет и имплементирует барьеры памяти.

  

---

  

## JMM: Atomicity

  

Важная часть JMM, которую я не упоминал ранее, это атомарность некоторых базовых действий. А именно:

  
- Чтения и записи *reference переменных* (ссылок) являются атомарными
- Чтения и записи *примитивов* (кроме long/double) являются атомарными
- Чтения и записи *long/double переменных*, помеченных как `volatile`, являются атомарными
  

Что же нам дают эти свойства в многопоточной среде? Нам гарантируется, что при shared чтении переменной мы увидим или значение по умолчанию (`0`, `false`, `null`), или полное консистентное значение, но не половинное значение. Даже если в переменную пишут одновременно несколько тредов, то мы увидим результат записи одного из них, но не будет такой ситуации, что чтение увидит первую половину битов из одной записи, а вторую половину из другой записи.

  

Но почему мы вообще могли бы прочитать половинное значение? Дело в том, что некоторые типы в языке имеют размер (в битах) больший, чем длина [машинного слова](https://en.wikipedia.org/wiki/Word_\(computer_architecture\)) процессора. Например, 32-х битный процессор оперирует словами по 32 бита, но тип long/double содержит 64 бита. Соответственно, языку требуется совершить 2 записи по 32 бит, чтобы полностью записать значение. Из JLS [§17.7. Non-Atomic Treatment of double and long](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.7):

  

> For the purposes of the Java programming language memory model, a single write to a non-volatile `long` or `double` value is treated as two separate writes: one to each 32-bit half. This can result in a situation where a thread sees the first 32 bits of a 64-bit value from one write, and the second 32 bits from another write.  
>   
> Writes and reads of volatile `long` and `double` values are always atomic.  
>   
> Writes to and reads of references are always atomic, regardless of whether they are implemented as 32-bit or 64-bit values.

JCStress имеет готовый тест для этого случая — [BasicJMM\_02\_AccessAtomicity.java](https://github.com/openjdk/jcstress/blob/fa0af79c39b010128f61cb7945149681721ce320/jcstress-samples/src/main/java/org/openjdk/jcstress/samples/jmm/basic/BasicJMM_02_AccessAtomicity.java#L77):

  
```java
@JCStressTest
@Outcome(id = "0",  expect = ACCEPTABLE,             desc = "Seeing the default value: writer had not acted yet.")
@Outcome(id = "-1", expect = ACCEPTABLE,             desc = "Seeing the full value.")
@Outcome(           expect = ACCEPTABLE_INTERESTING, desc = "Other cases are violating access atomicity, but allowed under JLS.")
@Ref("https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.7")
@State
public static class Longs {
   long v;

   @Actor
   public void writer() {
      v = 0xFFFFFFFF_FFFFFFFFL;
   }

   @Actor
   public void reader(J_Result r) {
      r.r1 = v;
   }
}
```
  

Результаты запуска теста оттуда же:

  
```
This test would yield interesting results on some 32-bit VMs, for example x86_32:
       RESULT        SAMPLES     FREQ       EXPECT  DESCRIPTION
           -1  8,818,463,884   70.12%   Acceptable  Seeing the full value.
  -4294967296      9,586,556    0.08%  Interesting  Other cases are violating access atomicity, but allowed u...
            0  3,747,652,022   29.80%   Acceptable  Seeing the default value: writer had not acted yet.
   4294967295         86,082   <0.01%  Interesting  Other cases are violating access atomicity, but allowed u...
```
  

Как видите, в некоторых случаях мы увидели неконсистентное состояние переменной. То есть мы наблюдали переменную прямо посередине записи — `writer` записал первую половину битов, но еще не успел записать вторую.

  

Один из способов обеспечить атомарность записи и чтения для long/double — это пометить переменную как `volatile`. Другой способ — это работать с переменной под монитором, который обеспечивает атомарность всех действий, выполняемых внутри `synchronized` блока. Замечу, что эти манипуляции необходимы только в том случае, если переменная шарится между тредами — для локальных переменных это не имеет смысла.

  

---

  

## JMM: final fields

  

JMM дает очень полезную гарантию порядка и видимости записей для `final` полей: если ссылка на создаваемый объект не утекла во время работы конструктора (так, что ее мог увидеть другой тред), то все остальные треды, которые увидели non-null ссылку на этот объект, *гарантированно прочитают актуальные значения всех внутренних `final` полей объекта* вне зависимости от того, была гонка при чтении ссылки или нет.

  

Из спеки JLS [§17.5. final Field Semantics](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.5):

  

> An object is considered to be *completely initialized* when its constructor finishes. A thread that can only see a reference to an object after that object has been completely initialized is guaranteed to see the correctly initialized values for that object's `final` fields.  
>   
> The usage model for `final` fields is a simple one: Set the `final` fields for an object in that object's constructor; and do not write a reference to the object being constructed in a place where another thread can see it before the object's constructor is finished. If this is followed, then when the object is seen by another thread, that thread will always see the correctly constructed version of that object's `final` fields.

Это очень сильная гарантия, которая полностью избавляет нас от проблем memory reordering при чтении состояния объекта. Обычно интерпретация этих свойств звучит как *Safe Initialization* (безопасная инициализация), которая по сути дает нам всегда безопасную пабликацию (safe publication).

  

Обычно под капотом эта гарантия имплементируется с помощью `StoreStore` + `LoadLoad` барьера памяти. Именно это и сказано в [JSR-133 Cookbook](https://gee.cs.oswego.edu/dl/jmm/cookbook.html):

  

> 1. Issue a `StoreStore` barrier after all stores but before return from any constructor for any class with a final field.
> 2. If on a processor that does not intrinsically provide ordering on indirect loads, issue a `LoadLoad` barrier before each load of a final field.
>   

Таким образом, вот так JVM создает объект с `final` полями:

  
```
Object _obj = <new>  // memory allocation
_obj.f = 5;    // write final field in constructor
[StoreStore]
obj = _obj;       // publish
```
  

Благодаря тому, что `StoreStore` барьер запрещает переупорядочивание store операций вокруг барьера, нам гарантируется, что после записи ссылки мы также записали и все поля из конструктора. Более того, `StoreStore` гарантирует видимость всех изменений до барьера, если читатель увидел запись, сделанную после барьера.

  

Читаем объект мы следующим образом:

  
```
Object _obj = obj; 
[LoadLoad]
r1 = _obj.f; // read final field
```
  

Здесь `LoadLoad` барьер требуется для того, чтобы процессор не переупорядочил чтение ссылки с чтениями полей объекта.

  

Однако как и в случае с `volatile`, эти барьеры не требуются там, где процессор уже дает необходимые гарантии. Например, таких барьеров точно не будет на x86.

  

Интересно, что благодаря такой имплементации, которая, например, используется в HotSpot JVM (см. [http://hg.openjdk.java.net/jdk/jdk/file/ee1d592a9f53/src/hotspot/share/opto/parse1.cpp#l1001](http://hg.openjdk.java.net/jdk/jdk/file/ee1d592a9f53/src/hotspot/share/opto/parse1.cpp#l1001)), нам неявно гарантируется видимость и всех остальных non- `final` полей. Однако это деталь имплементации, а не гарантия спеки, поэтому на это лучше не полагаться.

  

Семантика `final` полей напрямую касается иммутабельных объектов. Известно, что такие объекты можно безопасно шарить между тредами. Но без данной гарантии JMM это было бы не правдой, ведь проблема переупорядочивания все еще никуда не делась. Именно благодаря тому, что JMM автоматически берет на себя задачу по синхронизации `final` полей, мы имеем возможность корректно шарить иммутабельные объекты без использования примитивов синхронизации.

  

Давайте рассмотрим использование `final` полей на примере. Пусть мы имеем такой объект:

  
```java
public class Foo {
   private final int a; /* always visible */

   public Foo() {
      this.a = 5;
   }
}
public class Bar {
   private final int b; /* always visible */

   public Foo() {
      this.b = 7;
   }
}
public class DataHolder {
   private final Foo foo; /* always visible */
   private final int c; /* always visible */
   private Bar bar; /* may not be visible */
   private int d; /* may not be visible */

   public DataHolder() {
      this.foo = new Foo();
      this.bar = new Bar();
      this.c = 9;
      this.d = 10;
      /* StoreStore */
   }
}
```
  

Тогда мы имеем следующие гарантии — смотрите комментарии в коде:

  
```java
public class FinalFieldExample {

   private DataHolder instance;

   public void writer() {
      instance = new DataHolder();
   }

   public void reader() {
      DataHolder instance = this.instance; /* data race */
      /* LoadLoad */
      if (instance != null) {
         Foo foo = instance.foo; /* guaranteed to see non-null reference */
         int a = foo.a; /* guaranteed to see 5 */

         int c = instance.c; /* guaranteed to see 9 */

         Bar bar = instance.bar; /* no guarantee - may be null */
         if (bar != null) {
            int b = bar.b; /* guaranteed to see 7 */
         }

         int d = instance.d; /* no guarantee - may be 0 (default value) */
      }
   }
}
```
  

Интересные наблюдения:

  
- Хотя переменная `instance` и читается в гонке, но если мы увидели non-null ссылку, то нам гарантируется видимость всех внутренних `final` полей вне зависимости от наличия гонки
- Так как нам гарантируется видимость всех `final` полей, включая ссылки (reference variable), то по определению гарантируется и видимость `final` полей этих вложенных объектов. Это видно, например, по объекту `Foo`, который вложен в `DataHolder`
- На самом деле во всех местах, которые я пометил как `no guarantee`, я вам немного соврал. Как минимум на HotSpot JVM мы все равно прочитаем актуальные значения всех полей, так как все записи происходят до StoreStore барьера. Однако это деталь имплементации, а не гарантия языка
  

---

  

## Benign data races

  

Интересно, что наличие data race не всегда плохо, если это не влияет на корректность программы, а в некоторых случаях гонка даже является намеренной. Такие гонки называются **benign data race** (безопасная гонка).

  

Не будем далеко ходить за примером — взгляните на имплементацию [String#hashCode()](http://hg.openjdk.java.net/jdk/jdk/file/ee1d592a9f53/src/java.base/share/classes/java/lang/String.java#l1531) из OpenJDK (оригинальные комментарии в коде сохранены как есть):

  
```java
public final class String {
   /** Cache the hash code for the string */
   private int hash; // Default to 0

   /**
    * Cache if the hash has been calculated as actually being zero, enabling
    * us to avoid recalculating this.
    */
   private boolean hashIsZero; // Default to false;

   public int hashCode() {
      // The hash or hashIsZero fields are subject to a benign data race,
      // making it crucial to ensure that any observable result of the
      // calculation in this method stays correct under any possible read of
      // these fields. Necessary restrictions to allow this to be correct
      // without explicit memory fences or similar concurrency primitives is
      // that we can ever only write to one of these two fields for a given
      // String instance, and that the computation is idempotent and derived
      // from immutable state
      int h = hash;
      if (h == 0 && !hashIsZero) {
         h = isLatin1() ? StringLatin1.hashCode(value)
                 : StringUTF16.hashCode(value);
         if (h == 0) {
            hashIsZero = true;
         } else {
            hash = h;
         }
      }
      return h;
   }
}
```
  

Как видите, поля `hash` и `hashIsZero` не помечены как `volatile`, а соответственно:

  
1. Нет гарантии eventual visibility, то есть мы можем не увидеть записей в `hash` или `hashIsZero`
2. Даже если мы обнаружим запись, мы обнаружим ее в гонке и отношение happens-before между тредами не будет установлено
  

Однако из обоих ситуаций мы аккуратно восстанавливаемся.

  

В первой ситуации мы повторно вычисляем и записываем значения полей. Это валидно, так как результат вычисления `hashCode` остается неизменным для иммутабельного объекта (все строки в Java являются иммутабельными), то есть запись идемпотентна.

  

Вторую ситуацию мы обрабатываем путем чтения `hash` в локальную переменную, чтобы иметь только одну гонку вместо двух. Если мы единожды обнаружили запись и сохранили ее в локальную переменную, то она будет хранить это значение и при возвращении из метода.

  

Это очень тонкий момент, который может быть сложен для понимания. Представим, что мы убрали локальную переменную и сразу читаем shared переменную `hash`. Тогда даже если мы обнаружили запись на первом в порядке программы чтении `if (hash == 0)`, это не означает, что на последующем в порядке программы чтении `return hash` мы снова обнаружим эту запись. То есть, мы могли бы вернуть дефолтное значение из метода, что не корректно. Все дело в том, что оба чтения происходят в гонке. А как мы уже знаем, для действий в гонке не гарантируется никакого консистентного порядка и такие чтения могут быть переупорядочены как угодно (LoadLoad reordering). Формально с точки зрения JMM мы говорим так: первое чтение обнаружило запись в гонке и не установило happens-before между записью и чтением, а значит последующее чтение не обязано транзитивно увидеть ту же запись.

  

Сравните — без volatile валидны следующие выполнения:

  
```
write(hash, val) --race-> read(hash):val --hb-> read(hash): 0
write(hash, val) --race-> read(hash):val --hb-> read(hash): val
```
  

А с volatile валидно только такое выполнение:

  
```
write(hash, val) --hb-> read(hash):val --hb-> read(hash):val
```
  

Happens-before для действий в одном и том же треде здесь никак не помогает, потому что это *независимые чтения*. Как я уже говорил, happens-before не означает, что инструкции будут действительно выполняться в таком порядке, если это не нарушает happens-before.

  

И да, переменную `hashIsZero` мы тоже читаем в гонке. Но читаем всего один раз и не зависим от наличия hb между записью и чтением как в случае переменной `hash`, поэтому сохранение в локальную переменную не нужно.

  

Если вы хотите убедиться в том, что LoadLoad переупорядочивание возможно даже для одной и той же переменной, взгляните на данный jcstress тест — [https://github.com/blinky-z/JmmArticleHabr/blob/main/jcstress/tests/object/JmmReorderingObjectSameReadNullTest.java](https://github.com/blinky-z/JmmArticleHabr/blob/main/jcstress/tests/object/JmmReorderingObjectSameReadNullTest.java).

  

Как вы уже поняли, сделать правильно безопасную гонку нужно уметь. Если вы не поняли ничего из вышенаписанного или не уверены в своем случае, лучше ставьте везде где только можно `volatile` или `synchronized` — не ошибетесь.

  

---

  

## Заключение

  

Надеюсь, данная статья дала вам некоторое понимание JMM, а полученные знания помогут вам писать безопасные и корректные многопоточные программы.

  

Хотя я и привёл здесь много низкоуровневой информации, но на самом деле запоминать такие детали совершенно не обязательно — я лишь хотел дать вам более глубокое понимание того, что происходит под капотом JMM. Просто пользуйтесь предоставленными примитивами синхронизации, а JMM сделает все за вас, ведь она создана как раз с той целью, чтобы скрыть, абстрагировать нижние уровни и предоставить гарантии, избавляющие вас от проблем memory reordering.

  

Пользуйтесь JMM, и да пребудет с вами thread safety.

  

P.S: и запомните: **data races are evil**.

  

---

  

## Ресурсы

  

Обратите внимание на репозиторий в поддержку данной статьи — [https://github.com/blinky-z/JmmArticleHabr](https://github.com/blinky-z/JmmArticleHabr). Там вы сможете найти еще больше, не включенных в статью jcstress тестов и дизассемблированных программ, а также инструкции и результаты запуска тестов и дизассемблера на x86/arm64.

  

Основы:

  
- [Memory model](https://en.wikipedia.org/wiki/Memory_model_\(programming\)), Wikipedia
- [Memory ordering](https://en.wikipedia.org/wiki/Memory_ordering), Wikipedia
- [JLS §17.4. Memory Model](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html#jls-17.4), спецификация
- [The Java Memory Model](http://www.cs.umd.edu/~pugh/java/memoryModel/index.html) — "домашняя страница" JMM, содержащая кучу полезной информации
- [The JSR-133 Cookbook](https://gee.cs.oswego.edu/dl/jmm/cookbook.html) — неофициальный гайдлайн по имплементации JMM для разработчиков JDK, Doug Lea
- [Memory Barriers — a Hardware View for Software Hackers](https://raw.githubusercontent.com/tpn/pdfs/master/Memory%20Barriers%20-%20a%20Hardware%20View%20for%20Software%20Hackers%20\(July%2023%2C%202010\).pdf) — крайне подробный обзор работы CPU, кэша и барьеров памяти
- [How does memory reordering help processors and compilers?](https://stackoverflow.com/questions/37725497/how-does-memory-reordering-help-processors-and-compilers/37739933#37739933), StackOverflow
  

Memory Model:

  
- [Memory Consistency Models: A Tutorial](https://www.cs.utexas.edu/~bornholt/post/memory-models.html), blog post
- [Understanding the Java Memory Model](https://abailly.github.io/posts/jmm.html), blog post
- [Sequential consistency](https://en.wikipedia.org/wiki/Sequential_consistency), Wikipedia
- [The Synchronizes-With Relation](https://preshing.com/20130823/the-synchronizes-with-relation/), blog post
- [The Happens-Before Relation](https://preshing.com/20130702/the-happens-before-relation/), blog post
- [Acquire and Release Semantics](https://preshing.com/20120913/acquire-and-release-semantics/), blog post
- [An Introduction to Lock-Free Programming](https://preshing.com/20120612/an-introduction-to-lock-free-programming/#sequential-consistency), blog post
- [Memory Reordering Caught in the Act](https://preshing.com/20120515/memory-reordering-caught-in-the-act/), blog post
- [Sequential Consistency & Total Store Order](https://www.cis.upenn.edu/~devietti/classes/cis601-spring2016/sc_tso.pdf), slides
  

Блог Алексея Шипилева — это целая кладезь знаний про JMM и не только. Крайне советую прочитать следующие его статьи:

  
- [Java Memory Model Pragmatics](https://shipilev.net/blog/2014/jmm-pragmatics/), blog post
- [Safe Publication and Safe Initialization in Java](https://shipilev.net/blog/2014/safe-public-construction/), blog post
- [Close Encounters of The Java Memory Model Kind](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/), blog post
- [All Fields Are Final](https://shipilev.net/blog/2014/all-fields-are-final/), blog post
  

Compiler Memory Ordering:

  
- [Instruction scheduling](https://en.wikipedia.org/wiki/Instruction_scheduling), Wikipedia
- [Memory Ordering at Compile Time](https://preshing.com/20120625/memory-ordering-at-compile-time/), blog post
  

CPU Memory ordering/Memory Barrier:

  
- [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier), Wikipedia
- [Memory Barriers Are Like Source Control Operations](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/), blog post
- [Weak vs. Strong Memory Models](https://preshing.com/20120930/weak-vs-strong-memory-models/), blog post
- [Memory Barriers/Fences](https://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html?m=1), blog post
- [Linux kernel memory barriers](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/memory-barriers.txt), Linux kernel docs
- [Сводная таблица возможных переупорядочиваний среди различных микроархитектур](https://en.wikipedia.org/wiki/Memory_ordering#Runtime_memory_ordering)
- Intel (x86):  
	- [Intel Memory Ordering](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-system-programming-manual-325384.pdf#G13.14501), Intel docs
	- [Does an x86 CPU reorder instructions?](https://stackoverflow.com/questions/50307693/does-an-x86-cpu-reorder-instructions), StackOverflow
	- [Making sense of Memory Barriers](https://stackoverflow.com/questions/37798053/making-sense-of-memory-barriers), StackOverflow
	- [what is a store buffer?](https://stackoverflow.com/questions/11105827/what-is-a-store-buffer), StackOverflow
	- [Size of store buffers on Intel hardware? What exactly is a store buffer?](https://stackoverflow.com/questions/54876208/size-of-store-buffers-on-intel-hardware-what-exactly-is-a-store-buffer), StackOverflow
- ARM:  
	- [ARM Memory Ordering](https://developer.arm.com/documentation/den0024/a/Memory-Ordering?lang=en), ARM docs
  

CPU Cache:

  
- [CPU cache](https://en.wikipedia.org/wiki/CPU_cache), Wikipedia
- [Cache hierarchy](https://en.wikipedia.org/wiki/Cache_hierarchy), Wikipedia
- [Cache coherence](https://en.wikipedia.org/wiki/Cache_coherence), Wikipedia
- [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol), Wikipedia
- [CPU Cache Flushing Fallacy](https://mechanical-sympathy.blogspot.com/2013/02/cpu-cache-flushing-fallacy.html), blog post
- [Cache coherency primer](https://fgiesen.wordpress.com/2014/07/07/cache-coherency/), blog post
- [Memory barriers force cache coherency?](https://stackoverflow.com/questions/30958375/memory-barriers-force-cache-coherency), StackOverflow
  

Volatile:

  
- [Volatile](https://jpbempel.github.io/2012/10/09/volatile.html), blog post
- [Volatile and memory barriers](https://jpbempel.github.io/2015/05/26/volatile-and-memory-barriers.html), blog post
- [Java theory and practice: Managing volatility](http://web.archive.org/web/20210221170926/https://www.ibm.com/developerworks/java/library/j-jtp06197/), blog post by Brian Goetz
  

Книги:

  
- [A Primer on Memory Consistency and Cache Coherence](https://www.morganclaypool.com/doi/10.2200/S00962ED2V01Y201910CAC049)
- [Java Concurrency in Practice](https://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601)

+109

Будущее здесь: лучшее

Годнота из блогов компаний

Открыт приём в Школу анализа данных

Электроны зажигают!

Сезону Open source нужны твои истории

Дух горной прохлады

Космос ждёт

[Java разработчик](https://career.habr.com/vacancies/java_developer)

204 вакансии