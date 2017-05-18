---
layout: post
title: Java 实现线程超时
date: 2017-05-18
categories: thread
tags: [java]
description: Java实现线程超时机制
---

在线程执行或者RPC远程调用的时候，常常会需要配置超时属性。典型的场景为：A线程发起一个调用C，并等待调用结果返回，在超过一定时间阈值之后，抛出`TimeoutException`异常，并中断调用C。这里就隐含了几个条件。

1. 需要发起调用
2. 需要知道何时超时，发出超时信号
3. 进行超时处理，中断调用

根据经典的停机理论，一个线程是无法知道自己何时会终止。同样的，一个线程也无法知道自己何时会超时，要实现线程的超时机制肯定需要引入额外的线程，做超时检测和中断处理，具体代码如下。

````java
public class MainThread {

    private volatile Object response;

    private final Lock lock = new ReentrantLock();

    private final Condition done = lock.newCondition();

    private Thread thread;

    /**
     * async set
     *
     * @param response remove procedure response
     */
    public void setResponse(Object response) {
        lock.lock();
        this.response = response;
        done.signalAll();
        lock.unlock();
    }

    /**
     * simulation asynchronus call
     */
    private void asyncCall() {
        //调用线程
        thread = new Thread() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(5);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                setResponse(new Object());
            }
        };
        thread.start();
    }

    public Object get(int timeout) {

        asyncCall();

        long start = System.currentTimeMillis();

        if (!isDone()) {
            lock.lock();
            //noinspection InfiniteLoopStatement
            try {
                while (!isDone()) {
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        System.out.println("break");
                        break;
                    }
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            // 中断调用，抛出异常
            if (!isDone()) {
                thread.interrupt();
                throw new TimeOutExecption();
            }
        }
        return this.response;

    }

    public static void main(String[] args) {
        MainThread mainThread = new MainThread();
        //mainThread.get(6000);
        mainThread.get(4000);
		System.out.println("completed!!");
    }

    class TimeOutExecption extends RuntimeException {

    }

    public boolean isDone() {
        return response != null;
    }

}
````

其中`asyncCall()`方法实现独立线程调用，或者模拟远程方法调用，是方法执行的主体，`get(int timeout)`，为主线程获取方法返回值（调用成功），其中timeout为超时时间。

在样例代码中，使用了`ReentrantLock`和`Condition`完成线程等待和唤醒，原理也很简单，在主线程中，创建一个新的线程，并进行操作。主线程进行等待模式，使用`condition`的`await`方法进行等待，如果新线程在等待过程中正常完成操作，则会调用返回值设置，校验方法`isDone`变为true，如果超过了等待时间，并且`isDone`依然为false，则抛出`TimeOutException`异常，并新的线程。

