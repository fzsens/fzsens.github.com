## 为 Future 添加监听事件
在Java并发工具包`java.util.concurrent`中，高级的工具分成三类：`Executor Framework` 、并发集合`Concurrent Collection`以及同步器`Synchronizer`。其中的`Executor Framework` 通过控制`Thread`的启动，执行和关闭，简化了线程管理。内建对异步并发管理，允许线程异步返回。典型的`Executor`使用方法如下  

````java
ExecutorService service = Executors.newCachedThreadPool();
Future<String> future = service.submit(new Callable<String>() {
		public String call() throws Exception {
		  return "SUCCESS";
		}
	});
String v = future.get();
System.out.println(v);
service.shutdown();
````

- 创建线程池
- 提交执行任务(`Runnable`,或者为`Callable`)
- 调用`Future`的`get`方法，获得返回结果

线程并发的目的，主要在于提高CPU的利用率，从而提高程序性能。在我们创建和提交线程之后，经常需要在线程执行完毕之后进行一些回调操作，`Executor Framework`也提供了相应的处理机制。调用`sumbit`方法之后，会立即返回的`Future`表示异步计算的结果，使用`future.get()`会堵塞线程直到线程执行完毕将对应的结果返回(暂不考虑异常处理)。  
如果同事提交多个线程，并希望在每个线程线程执行完成之后立即能够执行对应的回调操作，就需要实时监控`future.get()`，最初的设想采用轮询操作,通过对Future进行实时遍历来处理回调，代码如下：  
````java
public static void main(String[] args) {
  ExecutorService service = Executors.newCachedThreadPool();
  List<Future<String>> futures = new ArrayList<Future<String>>();
  for (int i = 0; i < 5; i++) {
    final int index  = i;
    Future<String> future = service.submit(new Callable<String>() {
      public String call() throws Exception {
        //执行线程休眠操作
        if( index%2 == 0) {
          TimeUnit.SECONDS.sleep(5);
        }
        return "SUCCESS"+index;
      }
    });
    //添加入futures
    futures.add(future);
  }
  //调用shutdown() 会执行已经提交的线程，拒绝再提交新的线程
  service.shutdown();
  //轮询处理结果. *有严重的缺陷*
  while(!isAllDone(futures)) {
    //①
    for (int i = futures.size() -1 ; i >= 0 ; i --) {
      Future<String> future = futures.get(i);
      //②
      if(future.isDone()) {
        System.out.println(future.get());
        futures.remove(i);
      }
    }
    //③
  }
  
  }
  //轮询检查是否所有的Future都处理完毕
  public static boolean isAllDone(List<Future<String>> futures) {
  for (Future<String> future : futures) {
    //④
    if(!future.isDone()) {
      return false;
    }
  }
  return true;
}
````
上面的代码，使用了轮询来处理Future列表监控状态的变化，在`while`循环中，判断这个结果。但是这个方案存在一个重大的缺陷，及在执行完`③`到执行`④`之间的时间内，可能有一些Future 的状态变化为已经完成，则`isAllDone`测试为`ture`，循环会退出。因此这一些Future的返回结果无法被正确处理。  
因此在外部想要实时监控Future的状态存在一定的困难，想要避免`get()`方法的等待，就必须使用实时返回结构的`isDone()`来识别Future的执行情况，但是由于状态的处理和线程监控程序存在时间差异，因此很难保证结果的一致性，很容易因为状态的变化，而导致异常。  
如果能够让Future自己监控自己的执行结果，从而能够主动调用回调函数，那么这个问题就迎刃而解了。有一些类似C#中的`delegate`机制，通过委托调用，当发生特定事件时，就可以自动调用回调方法。  
1. 首先创建拓展Future,添加监听的功能，入参为回调方法和执行回调方法的线程执行器。
````java
/**
   * 拓展Future 增加监听行为
   *
   * @param <T>
   */
  public interface FutureListener<T> extends Future<T>{
    /**
     * 添加监听
     * @param listener
     * @param executor
     */
    void addListener(Runnable listener, Executor executor);
  }
````
2. 创建Futrue的状态代理
````java
/**
   * 
   * @author thierry.fu
   *
   * @param <V>
   */
  public abstract class AbstractFuture<V>  implements Future<V>{
    /**
     * 交由子类实现
     * @return
     */
    protected abstract Future<V> delegate();

    public boolean cancel(boolean mayInterruptIfRunning) {
      return delegate().cancel(mayInterruptIfRunning);
    }

    public boolean isCancelled() {
      return delegate().isCancelled();
    }

    public boolean isDone() {
      return delegate().isDone();
    }

    public V get() throws InterruptedException, ExecutionException {
      return delegate().get();
    }

    public V get(long timeout, TimeUnit unit) throws InterruptedException,
        ExecutionException, TimeoutException {
      return delegate().get(timeout, unit);
    }
    
  }
````
3. 创建包装类，在此处实现事件的监听和调用
````java
  /**
   * Future 包装类
   *
   * @param <T>
   */
  public  class FutureListenerWrapper<T> extends AbstractFuture<T> implements FutureListener<T> {
    
    //默认包装类监听程序执行器
    private static final Executor defaultWrapperExecutor = Executors.newCachedThreadPool();
    
    //实际Future
    private final Future<T> delegate;
    
    //监听程序执行器
    private final Executor adapterExecutor;
    
    //启动监听
    private final AtomicBoolean hasListeners = new AtomicBoolean(false);
    
    public FutureListenerWrapper(Future<T> delegate) {
      this(delegate,defaultWrapperExecutor);
    }
    
    public FutureListenerWrapper(Future<T> delegate,Executor adapterExecutor) {
      this.delegate = delegate;
      this.adapterExecutor = adapterExecutor;
    }

    public void addListener(Runnable listener, Executor executor) {
      final Runnable listenerInner = listener;
      final ExecutorService executoInner = (ExecutorService)executor;
      //设置监听
      if(hasListeners.compareAndSet(false, true)) {
        if(this.isDone()) {
          //如果已经返回则立刻执行回调函数
          executoInner.execute(listenerInner);
          executoInner.shutdown();
          return;
        }
        adapterExecutor.execute(new Runnable() {
          public void run() {
            try {
              //等待直到线程直接完成
              delegate.get();
            } catch (InterruptedException e) {
              e.printStackTrace();
            } catch (ExecutionException e) {
              e.printStackTrace();
            }
            executoInner.execute(listenerInner);
            executoInner.shutdown();
          }
        });
        ((ExecutorService)adapterExecutor).shutdown();
      }
    }

    @Override
    protected Future<T> delegate() {
      return this.delegate;
    }

  }
````
4. 测试方法
````java
  ExecutorService executor = Executors.newCachedThreadPool();
  ExecutorService callBackExecutor = Executors.newCachedThreadPool();
  Future<String> future = executor.submit(new Callable<String>() {
    public String call() throws Exception {
      TimeUnit.SECONDS.sleep(2);
      System.out.println("线程执行完成");
      return "SUCCESS";
    }
  });
  FutureListenerWrapper<String> futureWrapper  = new FutureListenerWrapper<String>(future);
  futureWrapper.addListener(new Runnable() {
    public void run() {
      System.out.println("回调方法调用");
    }
  },callBackExecutor);
  System.out.println("主线程执行完成");
  executor.shutdown();
````

整体的思路本质上就是重新启动线程来监听Future的执行结果，由代理对象来管理目标Future。上面的代码也是沿用这种设计方式。其中还存在对线程执行器管理的一些考虑不周。因此如果在实际工作中需要使用到Future监听可以使用Google的`Guava`实现的思路基本是一致的。