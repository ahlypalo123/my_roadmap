---
title: "Java concurrency in practice: Executors"
source: "https://medium.com/@svosh2/java-concurrency-in-practice-executors-e04396778041"
author:
  - "[[Kostiantyn Ivanov]]"
published: 2024-04-02
created: 2025-05-03
description: "A thread pool executor is a high-level concurrency framework in Java that manages and reuses a pool of worker threads to efficiently execute tasks concurrently. It abstracts the complexity of…"
tags:
  - "clippings"
---
## What thread pool executor is?

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*WHh21lY25oJ5Z2wbMLxR4w.png)

A thread pool executor is a high-level concurrency framework in Java that manages and reuses a pool of worker threads to efficiently execute tasks concurrently. It abstracts the complexity of managing individual threads and provides a more efficient way to handle multiple tasks in a multithreaded environment. Thread pool executors are part of the `java.util.concurrent` package and are commonly used in applications that require parallelism and concurrency.

## History

## Java 5 (J2SE 5.0 — September 2004):

- The `ThreadPoolExecutor` class was introduced in Java 5 as part of the `java.util.concurrent` package.

## Executors hierarchy and functionality

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*8Sl0p_LhYyp825IQWDvFUw.png)

## Executor

The `Executor` interface provides a simple and high-level way to decouple task submission from task execution, allowing you to execute tasks asynchronously. `Executor` interface defines a single method:

`void execute(Runnable command)`:  
submit a Runnable task for execution.

## ExecutorService

The `ExecutorService` interface is an extension of the `Executor` interface in Java's concurrency framework. It provides a more comprehensive and feature-rich way to manage and control the execution of tasks asynchronously. Here's an explanation of the `ExecutorService` interface methods:

`Future<?> submit(Runnable task)`:  
This method submits a `Runnable` task for execution and returns a `Future<?>` representing the result of the task. Since `Runnable` tasks don't produce a result, the returned `Future` is generally not used for retrieving a result but for tracking task completion and potential exceptions.

`<T> Future<T> submit(Callable<T> task)`:  
Similar to `submit(Runnable)`, this method submits a `Callable<T>` task for execution and returns a `Future<T>` that can be used to retrieve the result of the task when it completes.

`List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)`:  
This method submits a collection of `Callable` tasks for execution and waits until all tasks are completed. It returns a list of `Future<T>` objects, one for each submitted task, allowing you to retrieve the results or check for exceptions.

`List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)`:  
Similar to `invokeAll(Collection)`, this method submits a collection of `Callable` tasks for execution and waits for a specified amount of time for all tasks to complete. It returns a list of `Future<T>` objects and **allows you to specify a timeout** for task completion.

`<T> T invokeAny(Collection<? extends Callable<T>> tasks)`:  
This method submits a collection of `Callable` tasks for execution and waits until **at least one** of them completes successfully (without throwing an exception). It returns the result of the first successfully completed task and cancels the remaining tasks.

`<T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)`:  
Similar to `invokeAny(Collection)`, this method submits a collection of `Callable` tasks and waits for a specified amount of time for **at least one** of them to complete successfully. It returns the result of the first successfully completed task or throws a `TimeoutException` if none of the tasks completes **within the specified timeout**.

`void shutdown()`:  
This method initiates an orderly **shutdown** of the `ExecutorService`.  
The service **stops accepting new tasks** and **waits for previously submitted** tasks to complete. Once all tasks are completed, the `ExecutorService` is terminated.

`List<Runnable> shutdownNow()`:  
This method attempts to **stop all actively executing tasks, halts the processing of waiting tasks**, and returns a list of tasks that were waiting to be executed. It **forcibly terminates** the `ExecutorService`.

`boolean isShutdown()`:  
This method returns `true` if the `ExecutorService` has been shut down, either by invoking `shutdown()` or if it has completed its execution.

`boolean isTerminated()`:  
This method returns `true` if all tasks have completed after a shutdown request.

`boolean awaitTermination(long timeout, TimeUnit unit)`:  
This method blocks until all tasks have completed execution after a shutdown request or until the timeout occurs. It returns `true` if all tasks have completed within the specified timeout; otherwise, it returns `false`.

## ScheduledExecutorService

The `ScheduledExecutorService` interface in Java extends the `ExecutorService` interface and provides additional methods for scheduling tasks to run at a specific time or with fixed time intervals. It is commonly used for tasks that need to be executed periodically or with a delay. Here are the key methods provided by the `ScheduledExecutorService` interface:

`ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit)`:  
This method schedules a `Runnable` task to run after a specified delay. The `delay` parameter specifies the amount of time to wait before executing the task, and the `unit` parameter defines the time unit (e.g., seconds, milliseconds). It returns a `ScheduledFuture<?>` representing the pending execution of the task, allowing you to cancel the task or retrieve information about its execution status.

`<V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit)`:  
Similar to the previous method, this schedules a `Callable` task to run after a specified delay. It returns a `ScheduledFuture<V>` for tracking the task's execution and retrieving the result when it completes.

`ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)`:  
This method schedules a `Runnable` task to run at fixed-rate intervals, starting after an initial delay. The `initialDelay` parameter specifies the delay before the first execution, and the `period` parameter defines the time interval between subsequent executions. The `unit` parameter determines the time unit for both the initial delay and the period. The task will be executed at fixed intervals regardless of the task’s execution time.

`ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)`:  
This method schedules a `Runnable` task to run with a fixed delay between the end of one execution and the start of the next one. The `initialDelay` parameter specifies the delay before the first execution, and the `delay` parameter defines the time interval between executions. The `unit` parameter determines the time unit for both the initial delay and the delay between executions. The task will be executed with a delay between each execution, which means that the next execution doesn’t start until the previous one completes.

`ScheduledFuture<?> scheduleAtFixedRate(Runnable command, Instant startTime, long period, TimeUnit unit)`:  
This method schedules a `Runnable` task to run at fixed-rate intervals, starting at the specified `Instant` time. The `period` parameter defines the time interval between subsequent executions. The `unit` parameter determines the time unit for the period.

`ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, Instant startTime, long delay, TimeUnit unit)`:  
Similar to the previous method, this schedules a `Runnable` task with a fixed delay between executions, starting at the specified `Instant` time. The `delay` parameter defines the time interval between executions. The `unit` parameter determines the time unit for the delay.

## ThreadPoolExecutor

`ThreadPoolExecutor` is a class in the Java `java.util.concurrent` package that provides a highly flexible and configurable way to create and manage thread pools in Java applications. It serves as one of the most commonly used implementations of the `ExecutorService` interface and is a fundamental building block for managing concurrency and parallelism in Java programs.

The ThreadPoolExecutor class manages several important components:  
**Pool Size:** It allows you to specify the core pool size and maximum pool size. The core pool size is the number of threads kept alive even when they are idle, while the maximum pool size defines the maximum number of threads that can be created when the core threads are busy, and the work queue is full.  
**Work Queue:** The work queue holds tasks that are waiting to be executed. Tasks are queued here when there are no available threads to execute them.  
**Thread Factory:** You can provide a custom ThreadFactory to control the creation of threads in the pool.  
**Rejected Execution Handler:** This component handles tasks that cannot be executed, typically because the pool is full and the work queue is also full. Java provides several built-in policies (e.g., AbortPolicy, CallerRunsPolicy, DiscardPolicy, DiscardOldestPolicy) or you can create a custom handler.

`ThreadPoolExecutor` manages the lifecycle of threads, keeping core threads alive and potentially terminating idle threads when the load decreases (if specified). Threads are kept alive and reused, reducing the overhead of creating and destroying threads for each task.

## ScheduledThreadPoolExecutor

The `ScheduledThreadPoolExecutor` is a class in the Java `java.util.concurrent` package that extends `ThreadPoolExecutor` and provides specialized support for scheduling tasks to run at specified times or with fixed time intervals.

## ForkJoinPool

`ForkJoinPool` is a specialized implementation of the `ExecutorService` interface introduced in Java 7 as part of the `java.util.concurrent` package.

**Work-Stealing Algorithm:  
**`ForkJoinPool` uses a work-stealing algorithm to distribute and balance the workload among worker threads efficiently.

- In a work-stealing pool, each worker thread has its own deque (double-ended queue) of tasks to execute.
- When a worker thread finishes its own tasks, it can “steal” tasks from the end of another worker’s deque. This helps maintain high CPU utilization and minimizes thread contention.

**Divide-and-Conquer Parallelism:  
**`ForkJoinPool` is well-suited for solving problems using a divide-and-conquer approach, where a large task is split into smaller subtasks, each of which can be executed in parallel. The `ForkJoinTask` class is used to represent and manage these tasks, and it can be divided into subtasks using the `fork()` method.

**Joining Tasks:  
**The `join()` method of `ForkJoinTask` is used to wait for the completion of a task and obtain its result. When a task calls `join()` on another task, it blocks until the other task is completed, allowing for the orderly synchronization of tasks.

**Parallelism Control:  
**`ForkJoinPool` allows you to control the degree of parallelism by specifying the number of worker threads when creating the pool. You can create a `ForkJoinPool` with a specific parallelism level or use the default constructor, which creates a pool with a number of threads equal to the available processors on the system.

## Examples of usage

## ThreadPoolExecutor

To demonstrate usage ThreadPoolExecutor we will use a http server sources from previous [article](https://medium.com/@svosh2/java-concurrency-in-practice-threads-3d70664af2e2). Our code wll become much more simple implementing the same functionality.  
**Worker:**

```c
public class SimpleWorker implements Runnable {
    private final Socket clientSocket;
    public SimpleWorker(Socket clientSocket) {
        this.clientSocket = clientSocket;
    }

    @Override
    public void run() {
        processJob();
    }

    private void processJob() {
        try(OutputStream output = clientSocket.getOutputStream()) {
            StringBuilder response = new StringBuilder();
            String responseBody = "Hello from thread [" + Thread.currentThread().getId() + "]";
            addHeader(response, responseBody.length());
            response.append(responseBody);
            output.write(response.toString().getBytes());
            output.flush();

            clientSocket.close();
            Thread.sleep(10000);
            synchronized (this) {
                notifyAll();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    private void addHeader(StringBuilder sb, int contentLength) {
        sb.append("HTTP/1.1 200 OK\n")
            .append("Content-type: plain/text\n")
            .append("Content-length: ").append(contentLength)
            .append("\n")
            .append("\n");
    }
}
```

**ThreadPoolExecutorHttpServer:**

```c
public class ThreadPoolExecutorHttpServer {
    private static final ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(100);

    public static void main(String[] args) {
        new ThreadPoolExecutorHttpServer().start();
    }

    private void start() {
        int port = 8080;

        try {
            // Create a server socket on port 8080
            ServerSocket serverSocket = new ServerSocket(port);
            System.out.println("Server listening on port " + port);

            while (true) {
                // Accept incoming client connections
                Socket clientSocket = serverSocket.accept();
           
                System.out.println("Accepted connection from " + clientSocket.getInetAddress());
                EXECUTOR_SERVICE.submit(new SimpleWorker(clientSocket));

            }
        } catch (IOException e) {
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }
}
```

As you can see — all we need is to specify the executor service instance with thread amount limit and submit our workers for each incoming call.

Executors.newFixedThreadPool is the simple factory method where we have the next code:

```c
ThreadPoolExecutor(nThreads, nThreads,
                              0L, TimeUnit.MILLISECONDS,
                              new LinkedBlockingQueue<Runnable>())
```

So, we will handle up to 100 parallel calls and all the rest will be stored in this LinkedBlockingQueue. Let’s see how does it work. We will call our http server in 1000 parallel sessions.

We can notice pretty similar behaviour as in our own thread pool solution:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*aJ9ZUWCYZ98SYekf41neag.png)

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ZdgRoIolRT1-f7cPk6lF2g.png)

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*cE5oQdDwjzLcsMwyGgE5XA.png)

Probably the one confusing thing is a “park” state. No worries, it’s pretty much the same as “waiting”. You can read more about it [here](https://www.baeldung.com/java-lang-thread-state-waiting-parking).

## ScheduledThreadPoolExecutor

This implementation works under the hood of different schedulers implementation. For example you can find in a `org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler` which you use by `@Scheduled` annotation.

We will create something similar to this by ourselves:

We add a **simplified version of annotation:**

```c
@Retention(RetentionPolicy.RUNTIME)
@interface Scheduled {

}
```

It will be just a mark annotation, but for sure we can add here all the functionality(cron string, scheduler parameters, etc.)

Our **scheduled worker** looks like this:

```c
class ScheduledWorker {

    @Scheduled
    public void execute() {
        System.out.println("Hello from scheduler: [" + Thread.currentThread().getId() + "]");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

We emulate some logic with 2 seconds duration.

Our **scheduler infrastructure code**:

```c
public class Scheduler {

    private static final ScheduledExecutorService SCHEDULER =
        Executors.newSingleThreadScheduledExecutor();

    public static void main(String[] args) {

        Map<Method, Object> scheduledMethods = findScheduledMethods();

        scheduledMethods.forEach(
            (m,o) -> SCHEDULER.scheduleAtFixedRate(
                () -> {
                    try {
                        m.invoke(o);
                    } catch (Exception e) {
                        throw new RuntimeException(e);
                    }
                }, 0L, 10, TimeUnit.SECONDS
            )
        );

        while (true) {

        }
    }

    private static Map<Method, Object> findScheduledMethods() {
        //Here can be a package scan for all the needed classes. We will leave just one class
        // for simplicity

        Method[] methods = ScheduledWorker.class.getMethods();
        Map<Method, Object> result = new HashMap<>();
        for (Method method : methods) {
            if (method.getAnnotation(Scheduled.class) != null) {
                try {
                    result.put(
                        method,
                        ScheduledWorker.class.getDeclaredConstructor().newInstance()
                    );
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }

            }
        }
        return result;
    }
}
```

We create a new single thread executor. We search of a classes, that have schedule annotated methods and for each of them we create a scheduled job, that will be triggered once in a 10 seconds (in the real application it can be configured in annotation or some configuration file). In our job we just invoke the method was annotated.

Running our application we can see, that our extra thread is waiting for 10 seconds after which “runs the logic”:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*BSRqAOlNbodia9hI5oK39g.png)

## ForkJoinPool

There are a lot of real-world usages of fork join pool. It present under the hood of `ComplitableFuture` as well as works behind the scene of all the parallel operations like `Arrays.parallelSort()` or `Collection.parallelStream()`. Let’s review a usage of a custom ForkJoinPool for parallel streams:

```c
{
  List<Integer> data = new ArrayList<>();
  for (int i = 0; i < 10000; i++) {
    data.add(i);
  }
  List<Integer> integers = data.parallelStream().map(i -> makeAJob(i)).toList();
}

private static int makeAJob(Integer i) {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        return i * 2;
}
```

We have a list of Integers and we want to perform some time consuming operation on each of them (for example: save them into data base or send to some another service. But for simplicity we just leave a thread sleep), when we will call our “toList” the common ForkJoinPool will take its job and our jobs will be done in parallel:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*8GLC1GhYu1Fljk3Jlg5lgg.png)

All good, we have a thread pool limited by CPU cores and parallel processing. But what is a disadvantage of this process? Common ForkJoinPool will be used by default for all the parallel operations in our JVM. It means that our time consuming operations may affect not only the current flow but could make an extra latency for other flow because they will share the sane threads.

Using our own ForkJoinPool instance we can avoid this situation:

```c
ForkJoinPool forkJoinPool = new ForkJoinPool(100);
List<Integer> integers = forkJoinPool.submit(
    () -> data.parallelStream().map(i -> makeAJob(i)).toList()
).get();
```

When the parallel operation will be performed within separate ForkJoinPool — its threads will be used instead of commons once:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*VCLKg5782Gz9P_XFQEycuw.png)

In the same way we can use `ComplitableFuture` and all the custom `FutureTasks`

## Parallel streams and Stateful operations

Be careful to use parallel steams with stateful operations such as “sorted” or “distinct”.  
The main point you need to know:

- distinct will work fine with parallel streams, but it has to consume the **entire stream** before continuing and this may use a lot of memory.
- If the source of the items is an unordered collection (such as hashset) or the stream is `unordered()`, then `distinct` is not worried about ordering the output and thus will be efficient

**The original discussion on SOF:**

[https://stackoverflow.com/questions/53645037/will-parallel-stream-work-fine-with-distinct-operation](https://stackoverflow.com/questions/53645037/will-parallel-stream-work-fine-with-distinct-operation)

## Summary

Executors bring us a new level of concurrency abstractions. Using Executors implementation we still can implement parallel execution but our code will be much more elegant, short and simple to support.

## Links

**Sources with examples:**[https://github.com/sIvanovKonstantyn/java-concurrency](https://github.com/sIvanovKonstantyn/java-concurrency)

**Other articles of series:**## [Java concurrency in practice: threads](https://medium.com/@svosh2/java-concurrency-in-practice-threads-3d70664af2e2?source=post_page-----e04396778041---------------------------------------)

What thread is?

medium.com

[View original](https://medium.com/@svosh2/java-concurrency-in-practice-threads-3d70664af2e2?source=post_page-----e04396778041---------------------------------------)## [Java concurrency in practice: virtual threads](https://medium.com/@svosh2/java-concurrency-in-practice-virtual-threads-251f3e22c435?source=post_page-----e04396778041---------------------------------------)

What virtual thread is?

medium.com

[View original](https://medium.com/@svosh2/java-concurrency-in-practice-virtual-threads-251f3e22c435?source=post_page-----e04396778041---------------------------------------)

## More from Kostiantyn Ivanov

## Recommended from Medium

[

See more recommendations

](https://medium.com/?source=post_page---read_next_recirc--e04396778041---------------------------------------)