---
title: "Demystifying Java wait(), notify(), and join() methods for multithreading: An in-depth look"
source: "https://medium.com/@adam.rizk9/demystifying-java-wait-notify-and-join-methods-for-multithreading-an-in-depth-look-ffb43a514bbc"
author:
  - "[[Adam Rizk]]"
published: 2023-12-16
created: 2025-05-03
description: "The wait(), notify(), and join() methods in Java are used to make one thread wait until another thread has accomplished a certain task. Some learners may find it a bit confusing the difference…"
tags:
  - "clippings"
---
Get unlimited access to the best of Medium for less than $1/week.[Become a member](https://medium.com/plans?source=upgrade_membership---post_top_nav_upsell-----------------------------------------)

[

Become a member

](https://medium.com/plans?source=upgrade_membership---post_top_nav_upsell-----------------------------------------)

The wait(), notify(), and join() methods in Java are used to make one thread wait until another thread has accomplished a certain task. Some learners may find it a bit confusing the difference between these methods and when one is more appropriate than the other. In this article, we’ll take a detailed look at how these methods are used, and how they work under the hood. I won’t go over the code syntax since that’s widely documented, rather, we’ll examine the behavior and inner working of these thread synchronization methods.

For the sake of clarity, we’ll call the thread that calls wait() or join() the *waiting* thread. We’ll label the other thread that the waiter needs to synchronize with the *worker* thread. The *waiting* thread calls either the wait() or join() method to make it pause until the *worker* thread has executed in some capacity. This terminology will be used throughout this article.

## join()

When the waiting thread calls join(), it will pause until the worker thread has *terminated*. No additional logic is needed in the worker thread to ensure synchronization. The waiting thread will automatically resume once the worker thread terminates. Take a look at the snippet below:

```c
//Assume have a class Worker that extends the Thread class
//Currently in the waiting thread
Thread worker = new Worker();  
worker.start(); // start the worker thread 

try {
  worker.join(); // wait for the worker thread to finish
} catch (InterruptedException e) {
  // Handle exception if necessary
}

//This line is only reached after the worker thread terminates
System.out.println("The worker thread has terminated");
//This thread now continues executing
```

In the snippet above, we have two threads. The *waiting* thread creates and starts the *worker* thread. It then waits, or pauses until the worker thread has finished. The waiting thread then resumes execution. See the chart below to see when each thread in the snippet above is active:

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*F2OBjK1UVNaSerAe9dynGg.png)

Timeline of waiting and worker threads with join() method

This approach to thread synchronization has a key limitation: the waiting thread must *wait until the other thread has terminated* until it can resume! By default, the join() method does not allow the waiting and worker threads to be run simultaneously in a controlled manner. The waiting thread pauses until the worker thread either terminates or until the timeout period is reached — whichever occurs first.

Therefore, the join() method is a blunt instrument: it does not allow granular control of when each thread is active. join() provides no mechanism for the worker thread to resume the waiting thread programmatically. Since the worker thread must terminate for the waiting thread to resume, join() does not allow both the waiting and worker threads to run simultaneously (unless a timeout is reached).

## wait() and notify()

The wait() and notify() methods in Java allow the programmer to precisely define when the waiting thread will resume in relation to the worker thread. The worker thread need not terminate — it can continue running after waking up the waiting thread.

To understand this better, let’s contrast the wait() and notify() methods with join(). The join() method achieves thread synchronization by making the waiting thread pause until the worker thread has *fully executed* and *terminates*. In contrast, the wait() and notify() methods allow the waiting thread to pause until worker thread has *partially executed* and is still active. The programmer defines precisely the circumstances under which the waiting thread resumes. When the waiting thread calls wait(), the execution of that thread is suspended until another thread calls notify() on that same object. Therefore, the worker thread can wake up the waiting thread at will.

Before looking at wait() and notify() in more detail, it’s necessary to understand Java monitors and synchronized blocks. We won’t go deep into java monitors in this article, for now suffice to say that a monitor is essentially a Java object that is “owned” by one thread at a time and is used to enforce mutual exclusion in blocks of code. There is an exchange in ownership of a monitor between threads each time these methods are used. Some key facts to understand are:

- ==It is necessary for a thread to own a monitor as a prerequisite before it can invoke wait() on the monitor object. This is why the wait() call must be inside a synchronized block — to ensure the thread owns the monitor at the time wait() is invoked==
- Once a thread calls wait() on the monitor, it *gives up any claim to the monitor* after entering a paused state. The monitor can now be claimed by any other thread.
- Another thread, such as a worker thread, takes ownership of this monitor. Remember, a monitor is simply a Java object, and any thread can claim this monitor by entering a synchronized block on this object.
- Once another thread has taken ownership of the monitor (such as by entering a synchronized block on this monitor), it can call the notify() method on the monitor object to resume the waiting thread.
- Once the notify() method is called, the worker thread does not *immediately* give up the monitor. It continues owning the monitor until it exits the synchronized block.
- Once the worker thread releases the monitor *and* calls notify(), the waiting thread reclaims ownership of the monitor and resumes execution.
- wait() and notify() are instance methods. Therefore, if you simply call these methods as-is, it will implicitly use **this** object as the monitor (i.e. the object this method belong to). To use another object as the monitor, as we do in the snippet below, you must call these methods on that object with something like obj.wait() and obj.notify()

See the code block below:

Class Mainclass, which is run by the *waiting* thread:

```c
class Mainclass {
    public static void main(String[] args) {
        Mainclass main = new Mainclass();
        main.start();
    }

    public void start() {
        //This code is run by the 'waiting' thread
        Object monitorObj = new Object();
        Worker worker = new Worker(monitorObj); //Our worker object now has access to the same object monitorObj as this thread
        //Therefore, both the waiting thread and the worker thread have the same object to synchronize upon

        Thread thread = new Thread(worker);
        
        
        synchronized(monitorObj) { //Claiming monitorObj's monitor
            System.out.println("About to start the worker thread");
            thread.start(); //The worker thread has been started
            try {
                System.out.println("This thread is going to pause until another thread wakes it up");
                monitorObj.wait(); //This thread now pauses and releases the monitor. It can be claimed by any other thread
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("This thread has now resumed since Worker has called notify() and released the monitor");
    }
}
```

Class Worker, which is run by the *worker* thread:

```c
public class Worker implements Runnable {

    private Object monitor;

    public Worker(Object monitorObj) {
        monitor = monitorObj;
    }

    @Override
    public void run() {
        System.out.println("Waiting to acquire the monitor");
        synchronized(monitor) {
            System.out.println("About to wake up the waiting thread");
            monitor.notify();
        }
        //Now that we are out of the synchronized block, the Worker thread has released the monitor
        //Furthermore, notify() was invoked in the synchronized block above
        System.out.println("Now that the waiting thread has woken up, this Worker thread can continue running as usual");
        //The worker thread continues running and executes additional code that may be present here
    }

}
```

Output:

```c
About to start the worker thread
This thread is going to pause until another thread wakes it up
Waiting to acquire the monitor
About to wake up the waiting thread
Now that the waiting thread has woken up, this Worker thread can continue running as usual
This thread has now resumed since Worker has called notify() and released the monitor
```

As you can see, there is an exchange of the monitorObj to enable this thread synchronization: it is initially owned by the main thread, passed to the worker thread, and then passed back to the main thread. Of course, this requires both threads to have access to the same object.

Notice how the worker thread is able to wake up the waiting thread at-will and continue executing. This would not be possible with the join() method

## Summary

By now, you may realize that join() and wait() are more similar than you may have initially realized. In either case, a thread waits until another thread has *fulfilled some condition*

- With wait() the other thread must fulfill the condition of calling notify() and releasing the monitor
- With join() the other thread must fulfill the condition of terminating

The join() method is suitable when you want a thread to wait until another thread has wholly executed, while the wait() and notify() methods can be used in conjunction when more fine-grained thread synchronization is called for.

In fact, the join() method uses wait() internally according to the [official documentation](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html#join\(long,%20int\)):

> join(): This implementation uses a loop of this.wait calls conditioned on this.isAlive. As a thread terminates the this.notifyAll method is invoked. It is recommended that applications not use wait, notify, or notifyAll on Thread instances.

I hope you enjoyed this guide. I’m always happy to discuss, answer questions, and explore collaboration opportunities. Feel free to contact me using the links below:

- [https://linktr.ee/adam\_r](https://linktr.ee/adam_r)
- [Twitter](https://x.com/AdamR1776)