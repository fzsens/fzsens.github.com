---
layout: post
title: ThreadPoolExecutor运行原理 
date: 2018-08-17
categories: java
tags: [java]
description: 分析Java线程池实现类ThreadPoolExecutor执行过程
---

Java并发工具集（J.U.C)是开发中使用使用最多的功能之一，其主要的目的是简化Java并发程序的开发过程。其中使用最频繁的则要数线程池技术。还记得刚从事工作的时候，就参考《Thinking In Java》中的例子实现了在`ExecutorService`基础之上的文件并发处理程序，而且出乎我意料之外，现在还在生产环境上稳定运行。本文主要分析J.U.C中线程池的执行过程和工作原理，作为自己学习的一点总结，以下的版本基于JDK8进行分析。

首先线程池的核心功能在于使用可控数量的线程来执行一定数量的任务，可控数量的线程数量可以减少无谓的CPU调度开销，使用设计良好的API可以降低编写并发线程的难度。假如我们自己实现一个简陋的线程池，我们会怎么定义？

1. 应该有一个数据结构用于存储工作线程，这边可以使用`List`或者为了保证工作线程不重复添加使用`Set`
2. 应该有一个数据结构用于存储等待执行的的任务，一个朴素的思想是希望任务能够按照投递的顺序进行执行，因此可以采用`Queue`来存储
3. 工作线程需要从任务队列中获取投递到其中的任务，执行并返回。如果任务带有具体的返回值，希望能设计成为异步的，一边减少客户端的等待

以上三点就是我们要实现的简陋线程池的核心功能，下面看看具体的代码实现

首先针对执行结果异步返回，将其设计为回调方式，提交任务后返回一个回调接口，等到实际要使用的时候，再调用`get`方法返回任务执行结果

````java

public interface Callback<T> {

    void onSuccess(T result);

    void onFaild(Throwable t);

    public boolean isDone();

    T get() throws InterruptedException, ExecutionException;

}

public class FutureCallback<T> implements Callback<T>{

    private T result ;

    private Throwable failure;

    private final CountDownLatch latch = new CountDownLatch(1);

    @Override
    public void onSuccess(T result) {
        latch.countDown();
        this.result = result;
    }

    @Override
    public void onFaild(Throwable t) {
        latch.countDown();
        this.failure = t;
    }

    @Override
    public boolean isDone() {
        return this.latch.getCount() == 0L;
    }

    @Override
    public T get() throws InterruptedException, ExecutionException {
        latch.await();
        return doGet();
    }

    private T doGet() throws ExecutionException {
        if(failure != null){
            throw new ExecutionException(failure);
        }
        return result;
    }
}

````

代码实现非常简单，在`get`方法中使用`CountDownLatch`类堵塞等待返回值到来。

然后实现核心的线程池，遵循在分析阶段做的定义，具体实现如下

````java

public class Executors {

    private final BlockingQueue<CallbackTask> taskQueue;

    private final Set<Thread> workers;

    static Logger LOG = Logger.getLogger(Executors.class.getName());

    public Executors(int poolSize, int maxTasks) {

        taskQueue = new ArrayBlockingQueue<CallbackTask>(poolSize);

        workers = new HashSet<Thread>(maxTasks);

        for (int i = 0; i < poolSize; i++) {
            final int index = i;
            Thread t = new Thread() {
                @Override
                public void run() {
                    LOG.fine("thread " + index + " started.");
                    while (true) {
                        CallbackTask callbackTask = taskQueue.poll();
                        if (callbackTask != null) {
                            Callable<String> tCallable = callbackTask.task;
                            try {
                                callbackTask.callback.onSuccess(tCallable.call());
                            } catch (Exception e) {
                                callbackTask.callback.onFaild(e);
                            }
                        }
                    }

                }
            };
            t.start();
            workers.add(t);
        }
    }

    public FurtureCallback<String> submit(Callable<String> task) throws InterruptedException {
        FurtureCallback<String> callback = new FurtureCallback<>();
        this.taskQueue.put(new CallbackTask(task, callback));
        return callback;
    }

    static class CallbackTask {
        public Callable<String> task;
        public FurtureCallback<String> callback;

        public CallbackTask(Callable<String> task, FurtureCallback<String> callback) {
            this.task = task;
            this.callback = callback;
        }
    }
}

````

`workers`用于存储执行线程，为了简化代码，在线程池创建的时候，就执行实例化并启动线程，然后将其放入`workers`集合中，每一个执行线程在一个`while`循环体内，不停地尝试从任务队列`taskQueue`中获取任务，由于`ArrayBlockingQueue`的`poll`方法提供锁保护机制，因此一个任务不会被多个执行线程消费，当获取到任务后，直接在执行线程中运行任务的`call`方法，并根据执行结果，回写到`FutureCallback`中。当提交任务的时候，实例化一个`FutureCallback`对象，并和任务一起封装在`CallbackTask`中，以便提供异步返回的功能。最后调用的客户端得到`FutureCallback`实例，可以从中获得执行结果。

````java

    Executors executors = new Executors(4, 100);
    List<FurtureCallback<String>> callbackList = new ArrayList<>();
    for (int i = 0; i < 200; i++) {
        final int taskNo = i;
        Callable<String> task = new Callable<String>() {
            @Override
            public String call() throws Exception {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return "Task " + taskNo + " Thread: " + Thread.currentThread().getName();
            }
        };
        FurtureCallback<String> callback = executors.submit(task);
        callbackList.add(callback);
    }

    while(!callbackList.isEmpty()) {
        Iterator<FurtureCallback<String>> iterator = callbackList.iterator();
        while(iterator.hasNext()) {
            FurtureCallback<String> callback = iterator.next();
            if(callback.isDone()) {
                LOG.info(callback.get());
                iterator.remove();
            }
        }
    }
    LOG.info("Complete");
````

为了模型线程的处理，在每个待执行任务中等待一秒钟，为了限制系统系统使用的线程数量，我们采用4个线程来执行200个任务，并且线程池中最多同时放100个任务，由于`ArrayBlockingQueue`的特性，超出队列限制的提交动作会被堵塞直到队列有空闲，这可能并非一个最优的做法，后续在分析JUC中的线程池实现的时候，会看到它是采用了一个拒绝的策略。运行之后，可以看到大约在50s后线程池运行完毕所有的200个任务。到此在我们实现的这个简单的线程中，已经能够实现线程池的核心功能。接下来让我们分析一下JUC中线程池中的实现。

````java

    ExecutorService executor = new ThreadPoolExecutor(4, 8, 50, TimeUnit.SECONDS, new ArrayBlockingQueue<>(200));
    List<Future<String>> callbackList = new ArrayList<>();
        for (int i = 0; i < 200; i++) {
            final int taskNo = i;
            Callable<String> task = new Callable<String>() {
                @Override
                public String call() throws Exception {
                    try {
                        TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    return "Task " + taskNo + " Thread: " + Thread.currentThread().getName();
                }
            };
            Future<String> callback = executor.submit(task);
            callbackList.add(callback);
        }

        while(!callbackList.isEmpty()) {
            Iterator<Future<String>> iterator = callbackList.iterator();
            while(iterator.hasNext()) {
                Future<String> callback = iterator.next();
                if(callback.isDone()) {
                    LOG.info(callback.get());
                    iterator.remove();
                }
            }
        }
        LOG.info("Complete");
````

在这边采用`ThreadPoolExecutor`作为具体的线程池实例化对象，可以看到除了实现类之外，和我们自定义的线程池的运行时表现行为是一致的。下面我们以`ThreadPoolExecutor`类为起点，逐步分析JUC中线程池的实现机制。在我们选择的构造函数中使用了五个参数依次分别为

1. `corePoolSize` - 池中所保存的线程数，包括空闲线程。
2. `maximumPoolSize` - 池中允许的最大线程数。默认和`corePoolSize`相等
3. `keepAliveTime` - 当线程数大于核心时，到线程池终止前，多余的空闲线程等待新任务的最长时间，默认为 0。
4. `unit` - `keepAliveTime` 参数的时间单位。
5. `workQueue` - 执行前用于保持任务的队列。此队列仅保持由`execute`方法提交的`Runnable`任务，`submit`方法提交的任务，会被封装之后调用`execute`方法提交，默认为`LinkedBlockingQueue(Integer.MAX_VALUE)`。

此外还包括

6. `threadFactory` - 执行程序创建新线程时使用的工厂。默认为`Executors.DefaultThreadFactory`
7. `handler` - 由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。默认为`ThreadPoolExecutor.AbortPolicy`，当超出后抛出拒绝异常

这边就是`ThreadPoolExecutor`的主要核心对象，望文生义，我们也可以猜测到线程池内部的一些实现机制。参数1、2限制线程池数量，参数3，4用于空闲线程回收，参数5保存用户提交的任务，参数6是线程池初始化线程的工厂类，参数7用于任务队列满时候的处理，完整的构造函数如下

````java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
````

这里并没有直接初始化线程，而只是初始化了参数，与我们自己实现的线程池对比，显然这种延迟初始化的方式，对于资源的利用方面更胜一筹。接下来看看`ThreadPoolExecutor`的类继承结构

![ThreadPoolExecutor](_v_images/_threadpool_1534388286_22595.png) 

[图片来源](http://www.infoq.com/cn/articles/executor-framework-thread-pool-task-execution-part-01#)

`Executor`抽象定义了一个提交任务的执行器接口，只要用户将任务提交到`Executor`中就会运行，至于是怎么运行，由谁来运行，用户就可以不用关心了。
`ExecutorService`拓展了`Executor`，并添加了关闭线程管理（提交任务之后就相当于是将任务交给`Executor`这个执行器管理）`shutdown`，异步值返回`submit`，批量执行`invokeAll`等方法，进一步拓展了任务执行器的功能范围。
`AbstractExecutorService`提供了堆`ExecutorService`的默认实现，其中最核心的功能是将用户使用`submit`提交的任务，分封装为`FutureTask`，使其具备异步执行和返回的功能。

当调用`Future<String> callback = executor.submit(task);`向线程池中提交一个任务之后

````java

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

````

提交的`Callable`被封装入`FutureTask`中，这里的`FutureTask`与我们定义的`FutureCallback`有类似之处，不过`FutureCallback`值定义了异步获取返回值的方法，而`FutureTask`除了可以异步获取返回值之外，还定义了开始和取消计算的方法。并且与`FutureCallback`不同的是，为了能够取消正在执行的任务，`FutureTask`使用一个状态`state`变量用于表示任务的执行过程。

````java

    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

````

任务的初始状态为`NEW`，`COMPLETING`为运行完成到设置输出值的中间状态，当任务的执行线程被设置中断时状态修改为`INTERRUPTING`，当执行线程已经被中断时设置为`INTERRUPTED`，当执行线程被取消时设置为`CANCELLED`，当线程发生异常时则为`EXCEPTIONAL`，如果执行线程顺利完成任务的计算则状态设置为`NORMAL`，此时可以获取返回值。一个`FutureTask`的可能状态迁移路径如下

     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED

> 在JDK6的时候，`FutureTask`使用`AbstractQueuedSynchronizer`的同步器进行状态的控制，在JDK8中改为使用下面的状态机控制机制。

````java

    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            /**
            只有在set方法被执行后，才将执行线程设置为null，避免任务被并发执行
            **/
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            /**
            如果状态设置为中断，或者已经中断，则进行中断处理，
            handlePossibleCancellationInterrupt的逻辑为使用一个自旋锁，等待状态被设置为INTERRUPTED
            **/
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

````

执行的方法很简单，首先通过`UNSAFE`的`CAS`方法判断当前的任务是否正在被执行，如果尚未被执行，然后使用当前线程作为任务的执行线程，这个判断可以保证任务在不会被并发执行。直接调用`Callable`的`call`方法将其结果作为返回值，设置到`outcome`中，并调用`finishCompletion`完成运行。

>因为`Callable`本身即为带有返回值的任务，因此直接使用其方法的返回值作为计算结果即可，如果是`Runnable`则返回空，或者返回`submit`任务时候指定的返回值。在`FutureTask`中`Runnable`都会被使用`RunnableAdapter`适配器转换成为`Callable`进行执行。

````java
    public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
````

如果取消传入的参数为`true`，意味着直接中断任务的执行，则调用执行线程的`interrupt`方法，并将任务的状态设置为`INTERRUPTED`，中断之后会回到`run`方法中的异常处理。如果为`false`，则允许已经在运行的任务完成运算，并将状态设置为`CANCELLED`，如果这个任务尚未开始运算，则永远不会被执行。

````java

    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            /**抛出等待超时异常**/
            throw new TimeoutException();
        return report(s);
    }

````

获取最后的结果值，当`state`的状态小于等于`COMPLETING`，也就是任务尚未执行完成的时候，调用`awaitDone`进入等待状态，等待超时后抛出超时异常。

````java

    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
            /**短暂让出CPU执行时间**/
                Thread.yield();
            else if (q == null)
            /*创建一个等待节点，并入栈*/
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                /**等待超时,移除对应的等待节点**/
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }

````

在这边有一个`WaitNode`类，对应到`FutureTask`中的`private volatile WaitNode waiters;`，`waiters`中用了`Trieber Stack`来保存等待结果的线程。

>`Trieber Stack`是使用`CAS`技术的无锁并发栈，通过对已有栈的栈顶元素进行`CAS`比较，实现对出入栈的并发控制，`Trieber Stack`无法解决`ABA`问题，详细可参考[Wikipedia Trieber Stack](https://en.wikipedia.org/wiki/Treiber_Stack)

````java
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
````

当一个获取结果的线程（注意和执行线程区分开），调用`FutureTask`的`get`方法，会创建一个`WaitNode`节点`q`，并在调用`UNSAFE.compareAndSwapObject(this, waitersOffset, q.next = waiters, q)`，这段代码原子性地完成几件事情

1. 将当前的创建的节点`q`加入到`waiters`栈的栈顶，也就是将`q`的后继设置为原来的`waiters`
2. 判断当前的`waiters`未被修改，如果被修改则返回`false`再次进行重试
3. 用`q`作为当前的`waiters`

然后使用`LockSupport.park`进行等待状态，如果等待超时，则调用`removeWaiter`移除当前创建的`WaitNode`节点

````java

    private void removeWaiter(WaitNode node) {
        if (node != null) {
            /**当前node的thread属性设置为null**/
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                /**清除栈中,thread属性为null的节点**/
                    s = q.next;
                    if (q.thread != null)
                    /**当前节点还有，将pred设置为当前节点继续遍历，也就是pred存储的是上一个有效的节点**/
                        pred = q;
                    else if (pred != null) {
                    /**当前结果无效,并且之前的节点中存在有效的节点也就是pred，那么将当前节点的后继节点设置为pred节点的后继节点，也就是删除无效的当前节点**/
                        pred.next = s;
                        if (pred.thread == null) // check for race
                        /**如果前驱节点也无效了,则重新遍历**/
                            continue retry;
                    }
                    /**当前节点无效，并且在此节点前也不存在有效节点，则直接将waiters的栈顶节点（当前节点）删除，设置为后继节点**/
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

````

`removeWaiter`方法的主要作用是，清理无效的`WaitNode`节点，这些无效的节点主要由于中断或者超时导致的。

当任务最后执行完成的时候，就需要通过`waiters`来唤醒调用`Future.get()`等待任务执行结果的线程

````java

    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        done();
        callable = null;        // to reduce footprint
    }

````

`LockSupport.unpark(t)`方法，唤醒在等待执行结果的线程。并清理对应的节点。

分析完`FutureTask`之后，再继续返回到线程池中的`execute(ftask)`方法，在`ThreadPoolExecutor`中线程池接受到任务，并开始执行

在分析执行过程之前，首先看下`ThreadPoolExecutor`中对于线程池状态以及线程数量的处理策略

````

    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS; // 可以接受新的任务，正常处理任务队列里的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS; // 拒绝接收新的任务，正常可以处理任务队列里的任务
    private static final int STOP       =  1 << COUNT_BITS; // 拒绝接收新的任务，终止正在处理的工作进程，不处理任务队列中的任务
    private static final int TIDYING    =  2 << COUNT_BITS; // 所有的任务都已经终止，工作线程数量为0，并且很快会调用`terminated()`方法
    private static final int TERMINATED =  3 << COUNT_BITS; // `terminated()`调用完成

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
````

从这里可以看到，`ThreadPoolExecutor`使用原子变量`ctl`的前四位存储线程池的状态五个运行状态，后二十八位存储工作线程数量。当需要获取线程状态时候，调用`runStateOf(int c)`，当需要获取工作线程数量时，调用`workerCountOf(int c)`。

````java

    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

````

`execute`方法是`ThreadPoolExecutor`中执行任务的纲要

1. 首先判断当前的工作线程是否小于`corePoolSize`，如果是则调用`addWorker(command, true)`尝试添加一个核心工作线程
2. 如果已经超过核心线程数量，或者尝试添加核心线程失败，则判断线程池是否正在运行，以及任务队列是否可以容纳要添加的任务。如果允许添加入待执行的任务队列，则使用双重检查锁，再次确认线程池处于运行状态，如果在这个中间间隙中线程池被关闭，则拒接任务，并从任务队列中移除任务；如果线程池依然开启，则检测是否有活动的工作线程，如果没有启动一个不带有预定任务的线程。
3. 如果队列已满，则启动调用`addWorker(command, false)`尝试启动非核心工作线程，如果启动失败则拒绝任务。

从整个过程中可以看到`new ThreadPoolExecutor(4, 8, 50, TimeUnit.SECONDS, new ArrayBlockingQueue<>(200))`创建的线程池，最多可以添加208个任务，其中4个正在运行中，200个处于任务队列中。

````java

    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        ......// 第二部分
    }

````

`addWorker`主要的任务是添加并启动一个工作线程，第一部分，先判断线程的状态是否允许添加对应的工作线程，如果允许则调用`compareAndIncrementWorkerCount`增加工作线程数量，在这边同样使用`CAS`技术，避免在检测过程中工作线程数量的变化导致的不一致。

````java

     private boolean addWorker(Runnable firstTask, boolean core) {
        .......// 第一部分
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        /**新建一个工作线程**/
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                /**锁的目的主要用于保护workers添加的线程安全性**/
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                            /**校验状态，并添加到工作线程集合中**/
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                /**成功添加到workers中，执行工作线程**/
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
````

`addWorker`的第一部分实现检验并增加工作线程数量，第二部分则为实例化工作线程，添加到工作线程集合中，并调用`t.start()`运行工作线程，由于`t`在`Worker`中通过`getThreadFactory().newThread(this)`构造，实际上也就是调用`Worker`的`run`方法。如果添加失败，则减少工作线程的数量，并尝试终止线程池运行。


````java

    public void run() {
       runWorker(this);
    }

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
            /**锁定工作线程，在内部使用AQS实现，主要的作用是标记工作线程处于工作状态，避免数据竞争出现的不一致**/
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                    /**实际调用FutureTask的run方法**/
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    /**解除工作状态，工作线程可以被再次使用**/
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            /**如果getTask()返回null，跳出while循环后，执行工作线程的退出**/
            processWorkerExit(w, completedAbruptly);
        }
    }
````

首先执行`Worker`的初始任务，然后循环从任务队列中获取任务，并执行对应任务的`run`方法，注意这里实际上是对应的`FutureTask`中的`run`方法，并非是`start`，如果是`start`则是新启动一个线程，并不适用与这个场景，`Worker`本身就是一个可以独立运行的线程，无需在启动一个一对一的`FutrureTask`线程。

````java

    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                /**处于停止状态，并且任务队列为空**/
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                /**超过最大线程池数量，超时等其他的条件**/
                    return null;
                continue;
            }

            try {
            /**获取具体的任务**/
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

````

`getTask()`方法的主要作用是从工作队列中获取任务，如果其返回`null`，则会跳出`runWorker`中的`while`循环，并退出对应的工作线程。出现返回`null`的情况有如下几种

1. 工作线程数量超过`maximumPoolSize`
2. 线程池处于`STOP`状态
3. 线程池属于`SHUTDOWN`状态，并且任务队列为空
4. 工作线程等待任务超时，并且超时的工作线程满足：`allowCoreThreadTimeOut || workerCount > corePoolSize`，也就是核心工作线程允许超时回收或者当前工作线程数量大于核心线程数量；并且当任务队列不为空的时候，当前的工作线程不能线程池中的最后一个工作线程。也就是保证最少有一个工作线程可以执行任务队列中的任务。

````java

    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            /**如果是异常导致中断，则可能工作线程数量没有减少**/
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        /**保护workers的并发操作**/
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                /**若任务队列不为空，则最好要求有一个活跃的工作线程**/
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            /**确保有工作线程可以处理任务队列中的任务**/
            addWorker(null, false);
        }
    }

````

`processWorkerExit`主要的功能是回收工作线程，首先从`workers`集合中移除对应的工作线程；然后调用`tryTerminate()`尝试终止线程池；如果线程池的状态为`RUNNING`或者`SHUTDOWN`，还需要保证任务队列的任务会被顺利执行。

````java

    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                /**修改状态为TIDYING**/
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        /**唤醒调用了调用awaitTermination()的等待线程**/
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

````

在出现下面情况的时候，不做额外的处理直接返回
1. 如果线程池还在运行
2. 线程池处于`TIDYING`或者`TERMINATED`状态，说明任务线程已经都关闭了，不需要再处理
3. 线程池处于`SHUTDOWN`并且任务队列不为空，根据状态定义，还需要等待任务队列中的任务被处理完毕，才允许关闭线程池

````java

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                /**顺利tryLock之后，表明对应的工作线程已经没有任务正在运行属于空闲状态，可以安全中断**/
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                /**ONLY_ONE**/
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

````

`interruptIdleWorkers(ONLY_ONE)`每次只中断一个空闲的工作线程，并且中断的工作线程并非一定是调用`tryTerminate`的工作线程，而是`workers`队列中最新的工作线程，不过这并不影响最后所有的工作线程都会被一对一关闭。

最后当所有的`workers`中的工作线程都被回收，并且任务队列为空，线程池处于`SHUTDOWN`状态，则将线程池状态修改为`TIDYING`，并且调用`terminated()`默认实现是一个空方法，最后将线程池状态设置为`TERMINATED`并唤醒调用线程池`awaitTermination()`的等待线程。

````java

    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
````

`awaitTermination`等待`termination`竞态条件的唤醒

````java

    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            /**中断空闲的工作线程**/
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            /**中断所有的工作线程**/
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }

    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }

    void interruptIfStarted() {
        Thread t;
        /**getState() >= 0 表示线程正在运行中，中断Worker线程**/
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }

    private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        ArrayList<Runnable> taskList = new ArrayList<Runnable>();
        q.drainTo(taskList);
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }

````

`shutdownNow`线程池的关闭方法，首先尝试修改线程池状态为`STOP`，然后调用`interruptWorkers()`中断所有的工作线程，`drainQueue`将剩余未执行的任务，从任务队列中移除返回。

到此分析完`ThreadPoolExecutor`的整个运行过程，实际上这些代码数量并不多，比较复杂的部分在于各种状态的控制，和状态机的管理。除了对线程池的工作原理，知其然又知其所以然之外，对于自行编写复杂化的控制逻辑，这些方法可以作为借鉴和参考的例子。