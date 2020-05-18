---
title: "Java Thread 之 Daemon Thread"
date: 2020-04-25 11:29:00
author: "mingfer"
header-img: "img/post/bg-1.jpg"
catalog: true
tags: 
    - "Java"
    - "线程"
---

简单来说，Java 中的线程可以分为用户线程和守护（daemon）线程。用户线程就是 main 线程这样的工作线程；守护线程一般用于周期性的执行某个任务或等待某个事件的发生。

守护线程时一种低级别的线程，当 Java 虚拟机中仅剩下守护线程的时候 JVM 会退出。我们可以通过方法 `Thread.setDaemon(true)` 将线程设置为守护线程，该方法必须在 `Thread.start()`方法之前调用。在守护线程中创建的线程也是守护线程；守护线程可能因为没有用户线程而随时退出，所以不建议使用守护线程进行资源访问或者某些重要的定时任务。

下面是守护线程和用户线程在运行时候的区别。当线程池里面的线程是守护线程的时候，main 线程运行完退出的时候，JVM 会跟着退出，线程池中守护线程正在执行的任何也会被中断。而当线程池里面的线程是用户线程的时候，main 线程运行完之后，JVM 不会退出，线程池中的线程还在继续执行。

```java
package cn.mingfer.test;

import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class DaemonThreadTest {

    public static void main(String[] args) throws Exception {
//        userThread();
        daemonThread();
    }

    public static void userThread() throws Exception {
        Executors.newSingleThreadScheduledExecutor(runnable -> {
            Thread thread = new Thread(runnable,"test-user-thread");
            thread.setDaemon(false);
            return thread;
        }).scheduleWithFixedDelay(() -> { // 该任务会一直执行
            System.out.println("user thread executing.");
            try {
                Thread.sleep(1000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println("user thread executed.");
        }, 0, 1, TimeUnit.SECONDS);

        System.out.println("test daemon thread");
        Thread.sleep(2000);
        System.out.println("test daemon thread done");
    }

    public static void daemonThread() throws Exception {
        Executors.newSingleThreadScheduledExecutor(runnable -> {
            Thread thread = new Thread(runnable,"test-daemon-thread");
            thread.setDaemon(true);
            return thread;
        }).scheduleWithFixedDelay(() -> { // 该任务会因为 main 退出而中断
            System.out.println("daemon thread executing.");
            try {
                Thread.sleep(10000);
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println("daemon thread executed.");
        }, 0, 1, TimeUnit.SECONDS);

        System.out.println("test daemon thread");
        Thread.sleep(2000);
        System.out.println("test daemon thread done");
    }
}
```

`userThread()` 的运行结果： 

```
test daemon thread
user thread executing.
user thread executed.
test daemon thread done
user thread executing.
user thread executed.
user thread executing.
user thread executed.
user thread executing.
......
```

`daemonThread()` 的运行结果：

```
test daemon thread
daemon thread executing.
test daemon thread done

Process finished with exit code 0
```

