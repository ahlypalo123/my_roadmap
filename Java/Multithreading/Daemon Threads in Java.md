---
title: "Daemon Threads in Java"
source: "https://www.baeldung.com/java-daemon-thread"
author:
  - "[[Baeldung]]"
published: 2017-10-31
created: 2025-05-03
description: "Discover daemon threads in Java."
tags:
  - "clippings"
---
1. [1\. Overview](https://www.baeldung.com/#bd-overview)
2. [2\. Creating Daemon Threads](https://www.baeldung.com/#bd-creating-daemon-threads)
3. [3\. Differences Between Daemon and User Threads](https://www.baeldung.com/#bd-differences-between-daemon-and-user-threads)
4. [4\. Conclusion](https://www.baeldung.com/#bd-conclusion)

![announcement - icon](https://www.baeldung.com/wp-content/uploads/2022/04/announcement-icon.png)

Handling concurrency in an application can be a tricky process with many **potential pitfalls**. A solid grasp of the fundamentals will go a long way to help minimize these issues.

Get started with understanding multi-threaded applications with our **Java Concurrency** guide:

[\>> Download the eBook](https://www.baeldung.com/eBook-Java-Concurrency-NPI-1-Hgj18)

## 1\. Overview

Daemon threads in Java are background threads that support other tasks, like system [garbage collection](https://www.baeldung.com/jvm-garbage-collectors), logging, system monitoring, and more. **They run with lower priority and the [Java Virtual Machine (JVM)](https://www.baeldung.com/jvm-vs-jre-vs-jdk#jvm) terminates them after all user threads finish.** Many JVM threads are daemon threads by default.

In this short article, we’ll explore the main uses of daemon threads, and compare them to user threads. Additionally, we’ll demonstrate how to programmatically create, run, and verify if a thread is a daemon thread.

## 2\. Creating Daemon Threads

**To create daemon threads, all we need to do is to call *setDaemon()* of the [*Thread* API](https://www.baeldung.com/java-start-thread)***.* In this example, we’ll use the *NewThread* class which extends the *Thread* class:

```java
NewThread daemonThread = new NewThread();
daemonThread.setDaemon(true);
daemonThread.start();
```

**Any thread inherits the daemon status of the thread that created it.** Since the main thread is a user thread, any thread created inside the main method is by default a user thread.

The method *setDaemon()* can only be called after the *Thread* object has been created and the thread hasn’t been started. An attempt to call *setDaemon()* while a thread is running throws an *IllegalThreadStateException*:

```java
@Test(expected = IllegalThreadStateException.class)
public void whenSetDaemonWhileRunning_thenIllegalThreadStateException() {
    NewThread daemonThread = new NewThread();
    daemonThread.start();
    daemonThread.setDaemon(true);
}
```

**Additionally, we can use the *isDaemon()* method to check if a thread is a daemon thread**:

```java
@Test
public void whenCallIsDaemon_thenCorrect() {
    NewThread daemonThread = new NewThread();
    daemonThread.setDaemon(true);
    daemonThread.start();
    assertTrue(daemonThread.isDaemon());

    NewThread userThread = new NewThread();
    userThread.start();
    assertFalse(userThread.isDaemon());
}
```

## 3\. Differences Between Daemon and User Threads

So, we learned that Java offers two types of platform threads: user threads and daemon threads. **While user threads are high-priority threads, daemon threads are low-priority threads whose only role is to provide services to user threads.**

Let’s highlight other differences between daemon and user threads:

| Feature | Daemon Threads | User Threads |
| --- | --- | --- |
| Priority | Lower priority, mainly for background services | Higher priority, for main application tasks |
| JVM Behavior | JVM exits when only daemon threads are running | JVM continues running as long as any user thread is alive |
| Typical Use Cases | Garbage collection, system monitoring, etc. | Application logic, main program flow |
| Lifecycle | Automatically terminates when all user threads finish | Needs to be manually terminated |

Since daemon threads are meant to serve user threads and are only needed while they are running, they won’t prevent the JVM from exiting once all user threads have finished their execution.

Infinite loops typically exist in daemon threads and won’t cause problems. This is happening because any code, including the *finally* blocks, won’t be executed once all user threads have finished their execution. **As a result,****daemon threads aren’t recommended for I/O tasks.**

However, there are exceptions to this rule. **Poorly designed code in daemon threads can prevent the JVM from exiting.** For example, calling *Thread.join()* on a running daemon thread can block the application’s shutdown.

## 4\. Conclusion

In this quick tutorial, we’ve seen what daemon threads are and what they can be used for in a few practical scenarios. Following that, we compared daemon threads to user threads, focusing on their differences.

The code backing this article is available on GitHub. Once you're **logged in as a [Baeldung Pro Member](https://www.baeldung.com/members/)**, start learning and coding on the project.

![announcement - icon](https://www.baeldung.com/wp-content/uploads/2022/04/announcement-icon.png)

Handling concurrency in an application can be a tricky process with many **potential pitfalls**. A solid grasp of the fundamentals will go a long way to help minimize these issues.

Get started with understanding multi-threaded applications with our **Java Concurrency** guide:

[\>> Download the eBook](https://www.baeldung.com/eBook-java-concurrency-NPI-2-tGF65)