---
title: "Garbage Collection in Java – What is GC and How it Works in the JVM"
source: "https://www.freecodecamp.org/news/garbage-collection-in-java-what-is-gc-and-how-it-works-in-the-jvm/"
author:
  - "[[freeCodeCamp]]"
published: 2021-01-22
created: 2025-05-25
description: "By Siben Nayak In my previous article, I wrote about the Java Virtual Machine (JVM) and explained its architecture. As part of the Execution Engine component, I also briefly covered the Java Garbage Collector (GC). In this article, you will learn mor..."
tags:
  - "clippings"
---
By Siben Nayak

In my previous [article](https://www.freecodecamp.org/news/jvm-tutorial-java-virtual-machine-architecture-explained-for-beginners/), I wrote about the Java Virtual Machine (JVM) and explained its architecture. As part of the Execution Engine component, I also briefly covered the Java Garbage Collector (GC).

In this article, you will learn more about the Garbage Collector, how it works, and the various types of GC available in Java and their advantages. I will also cover some of the new experimental Garbage Collectors that are available in the latest Java releases.

## What is Garbage Collection in Java?

Garbage Collection is the process of reclaiming the runtime unused memory by destroying the unused objects.

In languages like C and C++, the programmer is responsible for both the creation and destruction of objects. Sometimes, the programmer may forget to destroy useless objects, and the memory allocated to them is not released. The used memory of the system keeps on growing and eventually there is no memory left in the system to allocate. Such applications suffer from " *memory leaks* ".

After a certain point, sufficient memory is not available for creation of new objects, and the entire program terminates abnormally due to OutOfMemoryErrors.

You can use methods like `free()` in C, and `delete()` in C++ to perform Garbage Collection. In Java, garbage collection happens automatically during the lifetime of a program. This eliminates the need to de-allocate memory and therefore avoids memory leaks.

Java Garbage Collection is the process by which Java programs perform automatic memory management. Java programs compile into bytecode that can be run on a Java Virtual Machine (JVM).

When Java programs run on the JVM, objects are created on the heap, which is a portion of memory dedicated to the program.

Over the lifetime of a Java application, new objects are created and released. Eventually, some objects are no longer needed. You can say that at any point in time, the heap memory consists of two types of objects:

- *Live* - these objects are being used and referenced from somewhere else
- *Dead* - these objects are no longer used or referenced from anywhere

The garbage collector finds these unused objects and deletes them to free up memory.

## How to Dereference an Object in Java

The main objective of Garbage Collection is to free heap memory by destroying the objects that don’t contain a reference. When there are no references to an object, it is assumed to be dead and no longer needed. So the memory occupied by the object can be reclaimed.

There are various ways in which the references to an object can be released to make it a candidate for Garbage Collection. Some of them are:

### By making a reference null

```java
Student student = new Student();
student = null;
```

### By assigning a reference to another

```java
Student studentOne = new Student();
Student studentTwo = new Student();
studentOne = studentTwo; // now the first object referred by studentOne is available for garbage collection
```

### By using an anonymous object

```java
register(new Student());
```

## How does Garbage Collection Work in Java?

Java garbage collection is an automatic process. The programmer does not need to explicitly mark objects to be deleted.

The garbage collection implementation lives in the JVM. Each JVM can implement its own version of garbage collection. However, it should meet the standard JVM specification of working with the objects present in the heap memory, marking or identifying the unreachable objects, and destroying them with compaction.

## What are Garbage Collection Roots in Java?

Garbage collectors work on the concept of *Garbage Collection Roots* (GC Roots) to identify live and dead objects.

Examples of such Garbage Collection roots are:

- Classes loaded by system class loader (not custom class loaders)
- Live threads
- Local variables and parameters of the currently executing methods
- Local variables and parameters of JNI methods
- Global JNI reference
- Objects used as a monitor for synchronization
- Objects held from garbage collection by JVM for its purposes

The garbage collector traverses the whole object graph in memory, starting from those Garbage Collection Roots and following references from the roots to other objects.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-76.png)

## Phases of Garbage Collection in Java

A standard Garbage Collection implementation involves three phases:

### Mark objects as alive

In this step, the GC identifies all the *live* objects in memory by traversing the object graph.

When GC visits an object, it marks it as accessible and thus alive. Every object the garbage collector visits is marked as alive. All the objects which are not reachable from GC Roots are garbage and considered as candidates for garbage collection.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-82.png)

### Sweep dead objects

After marking phase, we have the memory space which is occupied by live (visited) and dead (unvisited) objects. The sweep phase releases the memory fragments which contain these dead objects.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-83.png)

### Compact remaining objects in memory

The dead objects that were removed during the sweep phase may not necessarily be next to each other. Thus, you can end up having fragmented memory space.

Memory can be compacted after the garbage collector deletes the dead objects, so that the remaining objects are in a contiguous block at the start of the heap.

The compaction process makes it easier to allocate memory to new objects sequentially.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-85.png)

## What is Generational Garbage Collection in Java?

Java Garbage Collectors implement a *generational garbage collection strategy* that categorizes objects by age.

Having to mark and compact all the objects in a JVM is inefficient. As more and more objects are allocated, the list of objects grows, leading to longer garbage collection times. Empirical analysis of applications has shown that most objects in Java are short lived.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/ObjectLifetime.gif) *Source: oracle.com*

In the above example, the Y axis shows the number of bytes allocated and the X axis shows the number of bytes allocated over time. As you can see, fewer and fewer objects remain allocated over time.

In fact most objects have a very short life as shown by the higher values on the left side of the graph. This is why Java categorizes objects into generations and performs garbage collection accordingly.

The heap memory area in the JVM is divided into three sections:

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-70.png)

## Young Generation

Newly created objects start in the Young Generation. The Young Generation is further subdivided into:

- **Eden space** - all new objects start here, and initial memory is allocated to them
- **Survivor spaces (FromSpace and ToSpace)** - objects are moved here from Eden after surviving one garbage collection cycle.

When objects are garbage collected from the Young Generation, it is a *minor garbage collection event*.

When Eden space is filled with objects, a Minor GC is performed. All the dead objects are deleted, and all the live objects are moved to one of the survivor spaces. Minor GC also checks the objects in a survivor space, and moves them to the other survivor space.

Take the following sequence as an example:

1. Eden has all objects (live and dead)
2. Minor GC occurs - all dead objects are removed from Eden. All live objects are moved to S1 (FromSpace). Eden and S2 are now empty.
3. New objects are created and added to Eden. Some objects in Eden and S1 become dead.
4. Minor GC occurs - all dead objects are removed from Eden and S1. All live objects are moved to S2 (ToSpace). Eden and S1 are now empty.

So, at any time, one of the survivor spaces is always empty. When the surviving objects reach a certain threshold of moving around the survivor spaces, they are moved to the Old Generation.

You can use the `-Xmn` flag to set the size of the Young Generation.

## Old Generation

Objects that are long-lived are eventually moved from the Young Generation to the Old Generation. This is also known as Tenured Generation, and contains objects that have remained in the survivor spaces for a long time.

There is a threshold defined for the tenure of an object which decides how many garbage collection cycles it can survive before it is moved to the Old Generation.

When objects are garbage collected from the Old Generation, it is a *major garbage collection event*.

You can use the `-Xms` and `-Xmx` flags to set the size of the initial and maximum size of the Heap memory.

Since Java uses generational garbage collection, the more garbage collection events an object survives, the further it gets promoted in the heap. It starts in the young generation and eventually ends up in the tenured generation if it survives long enough.

Consider the following example to understand the promotion of objects between spaces and generations:

When an object is created, it is first put into the **Eden space** of the **young generation**. Once a minor garbage collection happens, the live objects from **Eden** are promoted to the **FromSpace**. When the next minor garbage collection happens, the live objects from both **Eden** and **FromSpace** are moved to the **ToSpace**.

This cycle continues for a specific number of times. If the object is still used after this point, the next garbage collection cycle will move it to the **old generation** space.

## Permanent Generation

Metadata such as classes and methods are stored in the Permanent Generation. It is populated by the JVM at runtime based on classes in use by the application. Classes that are no longer in use may be garbage collected from the Permanent Generation.

You can use the `-XX:PermGen` and `-XX:MaxPermGen` flags to set the initial and maximum size of the Permanent Generation.

## MetaSpace

Starting with Java 8, the **MetaSpace** memory space replaces the **PermGen** space. The implementation differs from the PermGen and this space of the heap is now automatically resized.

This avoids the problem of applications running out of memory due to the limited size of the PermGen space of the heap. The Metaspace memory can be garbage collected and the classes that are no longer used can be automatically cleaned when the Metaspace reaches its maximum size.

![](https://www.youtube.com/watch?v=X1DkoRGVRp4)

## Types of Garbage Collectors in the Java Virtual Machine

Garbage collection makes Java memory efficient because it removes the unreferenced objects from heap memory and makes free space for new objects.

The Java Virtual Machine has eight types of garbage collectors. Let's look at each one in detail.

## Serial GC

This is the simplest implementation of GC and is designed for small applications running on single-threaded environments. All garbage collection events are conducted serially in one thread. Compaction is executed after each garbage collection.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-68.png)

When it runs, it leads to a "stop the world" event where the entire application is paused. Since the entire application is frozen during garbage collection, it is not recommended in a real world scenario where low latencies are required.

The JVM argument to use the Serial Garbage Collector is `-XX:+UseSerialGC`.

## Parallel GC

The parallel collector is intended for applications with medium-sized to large-sized data sets that are run on multiprocessor or multithreaded hardware. This is the default implementation of GC in the JVM and is also known as Throughput Collector.

Multiple threads are used for minor garbage collection in the Young Generation. A single thread is used for major garbage collection in the Old Generation.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-66.png)

Running the Parallel GC also causes a "stop the world event" and the application freezes. Since it is more suitable in a multi-threaded environment, it can be used when a lot of work needs to be done and long pauses are acceptable, for example running a batch job.

The JVM argument to use the Parallel Garbage Collector is `-XX:+UseParallelGC`.

## Parallel Old GC

This is the default version of Parallel GC since Java 7u4. It is same as Parallel GC except that it uses multiple threads for both Young Generation and Old Generation.

The JVM argument to use Parallel Garbage Collector is `-XX:+UseParallelOldGC`.

## CMS (Concurrent Mark Sweep) GC

This is also known as the concurrent low pause collector. Multiple threads are used for minor garbage collection using the same algorithm as Parallel. Major garbage collection is multi-threaded, like Parallel Old GC, but CMS runs concurrently alongside application processes to minimize “stop the world” events.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-67.png)

Because of this, the CMS collector uses more CPU than other GCs. If you can allocate more CPU for better performance, then the CMS garbage collector is a better choice than the parallel collector. No compaction is performed in CMS GC.

The JVM argument to use Concurrent Mark Sweep Garbage Collector is `-XX:+UseConcMarkSweepGC`.

## G1 (Garbage First) GC

G1GC was intended as a replacement for CMS and was designed for multi-threaded applications that have a large heap size available (more than 4GB). It is parallel and concurrent like CMS, but it works quite differently under the hood compared to the older garbage collectors.

Although G1 is also generational, it does not have separate regions for young and old generations. Instead, each generation is a set of regions, which allows resizing of the young generation in a flexible way.

It partitions the heap into a set of equal size regions (1MB to 32MB – depending on the size of the heap) and uses multiple threads to scan them. A region might be either an old region or a young region at any time during the program run.

After the mark phase is completed, G1 knows which regions contain the most garbage objects. If the user is interested in minimal pause times, G1 can choose to evacuate only a few regions. If the user is not worried about pause times or has stated a fairly large pause-time goal, G1 might choose to include more regions.

Since G1GC identifies the regions with the most garbage and performs garbage collection on that region first, it is called Garbage First.

![Image](https://www.freecodecamp.org/news/content/images/2021/01/image-88.png)

Apart from the Eden, Survivor, and Old memory regions, there are two more types of regions present in the G1GC:

- *Humongous* - used for large size objects (larger than 50% of heap size)
- *Available* - the unused or non-allocated space

The JVM argument to use the G1 Garbage Collector is `-XX:+UseG1GC`.

## Epsilon Garbage Collector

Epsilon is a do-nothing (no-op) garbage collector that was released as part of JDK 11. It handles memory allocation but does not implement any actual memory reclamation mechanism. Once the available Java heap is exhausted, the JVM shuts down.

It can be used for ultra-latency-sensitive applications, where developers know the application memory footprint exactly, or even have (almost) completely garbage-free applications. Usage of the Epsilon GC in any other scenario is otherwise discouraged.

The JVM argument to use the Epsilon Garbage Collector is `-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`.

## Shenandoah

Shenandoah is a new GC that was released as part of JDK 12. Shenandoah’s key advantage over G1 is that it does more of its garbage collection cycle work concurrently with the application threads. G1 can evacuate its heap regions only when the application is paused, while Shenandoah can relocate objects concurrently with the application.

Shenandoah can compact live objects, clean garbage, and release RAM back to the OS almost immediately after detecting free memory. Since all of this happens concurrently while the application is running, Shenandoah is more CPU intensive.

The JVM argument to use the Epsilon Garbage Collector is `-XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC`.

## ZGC

ZGC is another GC that was released as part of JDK 11 and has been improved in JDK 12. It is intended for applications which require low latency (less than 10 ms pauses) and/or use a very large heap (multi-terabytes).

The primary goals of ZGC are low latency, scalability, and ease of use. To achieve this, ZGC allows a Java application to continue running while it performs all garbage collection operations. By default, ZGC uncommits unused memory and returns it to the operating system.

Thus, ZGC brings a significant improvement over other traditional GCs by providing extremely low pause times (typically within 2ms).

![Image](https://www.freecodecamp.org/news/content/images/2021/01/figure2_600w.jpg) *Source: oracle.com*

The JVM argument to use the Epsilon Garbage Collector is `-XX:+UnlockExperimentalVMOptions -XX:+UseZGC`.

**Note:** Both Shenandoah and ZGC are planned to be made production features and moved out of the experimental stage in JDK 15.

## How to Select the Right Garbage Collector

If your application doesn't have strict pause-time requirements, you should just run your application and allow the JVM to select the right collector.

Most of the time, the default settings should work just fine. If necessary, you can adjust the heap size to improve performance. If the performance still doesn't meet your goals, you can modify the collector as per your application requirements:

- **Serial** - If the application has a small data set (up to approximately 100 MB) and/or it will be run on a single processor with no pause-time requirements
- **Parallel** - If peak application performance is the priority and there are no pause-time requirements or pauses of one second or longer are acceptable
- **CMS/G1** - If response time is more important than overall throughput and garbage collection pauses must be kept shorter than approximately one second
- **ZGC** - If response time is a high priority, and/or you are using a very large heap

## Advantages of Garbage Collection

There are multiple benefits of garbage collection in Java.

First of all, it makes your code simple. You don’t have to worry about proper memory assignment and release cycles. You just stop using an object in your code, and the memory it is using will be automatically reclaimed at some point.

Programmers working in languages without garbage collection (like C and C++) must implement manual memory management in their code.

It also makes Java memory-efficient because the garbage collector removes the unreferenced objects from heap memory. This frees the heap memory to accommodate new objects.

While some programmers argue in favour of manual memory management over garbage collection, garbage collection is now a standard component of many popular programming languages.

For scenarios in which the garbage collector is negatively impacting performance, Java offers many options for tuning the garbage collector to improve its efficiency.

## Garbage Collection Best Practices

### Avoid Manual Triggers

Besides the basic mechanisms of garbage collection, one of the most important points to understand about garbage collection in Java is that it is non-deterministic. This means that there is no way to predict when garbage collection will occur at run time.

It is possible to include a hint in the code to run the garbage collector with the `System.gc()` or `Runtime.gc()` methods, but they provide no guarantee that the garbage collector will actually run.

### Use Tools for Analysis

If you don’t have enough memory to run your application, you will experience slowdowns, long garbage collection times, "stop the world" events, and eventually out of memory errors. This can indicate that your heap is too small, but can also mean that you have a memory leak in your application.

You can get help from a monitoring tool like `jstat` or *Java Flight Recorder* to see if the heap usage grows indefinitely, which might indicate a bug in your code.

### Default Settings are Good

If you are running a small, standalone Java application, you will most probably not need any kind of garbage collection tuning. The default settings should work just fine.

### Use JVM Flags for Tuning

The best approach to tuning Java garbage collection is setting flags on the JVM. Flags can adjust the garbage collector to be used (for example Serial, G1, and so on), the initial and maximum size of the heap, the size of the heap sections (for example, Young Generation, Old Generation), and more.

### Select the Right Collector

The nature of the application being tuned is a good initial guide to the settings. For example, the Parallel garbage collector is efficient but will frequently cause “stop the world” events, making it better suited for backend processing where long pauses for garbage collection are acceptable.

On the other hand, the CMS garbage collector is designed to minimize pauses, making it ideal for web based applications where responsiveness is important.

![](https://www.youtube.com/watch?v=4sBhc-pSILs)

## Conclusion

In this article, we discussed Java Garbage Collection, how it works, and its various types.

For many simple applications, Java garbage collection is not something that a programmer needs to consciously consider. However, for programmers who want to advance their Java skills, it is important to understand how Java garbage collection works.

This is also a very popular interview question, both at junior and senior levels for backend roles.

Thank you for staying with me so far. Hope you liked the article. You can connect with me on [LinkedIn](https://www.linkedin.com/in/theawesomenayak/) where I regularly discuss technology and life. Also take a look at some of and my [YouTube channel](https://www.youtube.com/channel/UCmWAaPgfWAkl-Jep5mY-NNg?sub_confirmation=1). Happy reading. 🙂