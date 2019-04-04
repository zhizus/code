---
layout: post
title:  "Java线程池详解"
date:  2016-09-03
categories: [并发编程]
keywords: java线程池,无界队列,饱和策略
---
## 1.概述

{% note%}
好的软件设计不建议手动创建和销毁线程。线程的创建和销毁是非常耗 CPU 和内存的，因为这需要 JVM 和操作系统的参与。64位 JVM 默认线程栈是大小1 MB。这就是为什么说在请求频繁时为每个小的请求创建线程是一种资源的浪费。线程池可以根据创建时选择的策略自动处理线程的生命周期。重点在于：在资源（如内存、CPU）充足的情况下，线程池没有明显的优势，否则没有线程池将导致服务器奔溃。有很多的理由可以解释为什么没有更多的资源。例如，在拒绝服务（denial-of-service）攻击时会引起的许多线程并行执行，从而导致线程饥饿（thread starvation）。除此之外，手动执行线程时，可能会因为异常导致线程死亡，程序员必须记得处理这种异常情况。
{% endnote %}
即使在你的应用中没有显式地使用线程池，但是像 Tomcat、Undertow这样的web服务器，都大量使用了线程池。所以了解线程池是如何工作的，怎样调整，对系统性能优化非常有帮助。
<!-- More-->
## 2.线程池的基础知识
### 2.1线程池的构造方法

Java提供了Executor框架提供了一套完善的多线程编程api，包括对线程生命周期，统计信息，程序管理机制，性能监控等功能。

``` java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

- `corePoolSize`：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者`prestartCoreThread`()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；

- `maximumPoolSize`：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用，直到线程池中的线程数不大于corePoolSize，即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的线程数为0；
- `unit`：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性：
- `workQueue`：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择：

```
ArrayBlockingQueue;//
LinkedBlockingQueue;//链式结构的阻塞队列，作为Executors.newFixThreadPool()生成线程池的默认队列，入列出列快，但是同时会产生Node对象
SynchronousQueue;//同步队列，没有存储能力，作为Executors.newCacheThreadPool()线程池的默认队列
```

- `threadFactory`：线程工厂，主要用来创建线程；
- `handler`：表示当拒绝处理任务时的策略，有以下四种饱和策略：

```java
ThreadPoolExecutor.AbortPolicy://丢弃任务并抛出RejectedExecutionException异常。
ThreadPoolExecutor.DiscardPolicy：//也是丢弃任务，但是不抛出异常。
ThreadPoolExecutor.DiscardOldestPolicy：//丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
ThreadPoolExecutor.CallerRunsPolicy：//由调用线程处理该任务
```

### 2.2 线程池的整体流程

了解完线程池的参数，下面这张线程架构图能够很好的帮助我们深入理解Java的线程池：

![](/images/threadPoolFramworkNew.png)

**ExecuteService简单的理解可以认为是：**

1.通过消费者模式来解耦线程资源和任务。（消费者模式真的是非常有用的模式，java的框架中大量使用到（logback也有使用，一个线程作为消费者来写日志，多个生产者将日志放入队列），只要涉及到需要解耦的可能都会考虑这个设计模式）

2.使用池的技术来管理稀缺资源，包括管理生命周期，监控等等



下面，借用[Qcon](http://www.infoq.com/cn/articles/java-threadPool)一张流程图来说说线程池的处理流程：

![](/images/threadPoolFlow.jpg)



**1.首先线程池判断基本线程池是否已满？没满，创建一个工作线程来执行任务。满了，则进入下个流程。**

**2.其次线程池判断工作队列是否已满？没满，则将新提交的任务存储在工作队列里。满了，则进入下个流程。**

**3.最后线程池判断整个线程池是否已满？没满，则创建一个新的工作线程来执行任务，满了，则交给饱和策略来处理这个任务。**



### 2.3如何创建一个线程池

我们可以通过构造函数初始化一个自己想要的线程池，但是一般不推荐这么做，除非Executors工厂方法构造的线程池不能满足我们的要求。
下面介绍一下Executos的api创建的线程池的特点：

``` java

public class Executors {

   /**
   *创建一个固定大小的线程池，每当提交一个任务就创建一个线程，知道达到线程池的最大数量，这时线程规模不在变化
   （如果某个线程发生了未知Exception，那么线程池将补充一个线程）
   **/
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    /**
    *创建一个可缓存的线程池，如果线程当前规模超过处理的需求，那么回收空闲线程（默认60s回收一次），
    当需求增加时，则可以添加新的线程，线程池的规模不存在任何限制
    **/
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }


    /**
    *是一个单线程的Executors，如果这个线程意外结束，则会创建另一个线程替代，newSingleThreadExecutor能保证任
    务串行执行，例如（FIFO，LIFO，优先级）
    *Netty中的NioEventLoop就是一个单线程的Executors，保证业务逻辑串行无锁执行
    **/
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

    /**
    *创建定时执行的线程池，功能类似Timer，建议使用newSingleThreadScheduledExecutor来取代Timer（
    1.Timer异常处理糟糕；2.Timer只会创建一个线程，不能保证定时的精确性）
    **/
    public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }

}
```

### 2.3对比newFixedThreadPool&newCachedThreadPool

了解newFixedThreadPool&newCachedThreadPool的区别有助于我们更好的理解和使用线程池，那么这两者到底有什么不同呢？

默认newCachedThreadPool会创建一个corePoolSize为0，maximumPoolSize为Integer.maxValue，队列为SynchronousQueue的线程池，默认60s回收一次空闲线程。这意味着newCachedThreadPool创建的线程池能够自动适应，有效的避免了由于任务依赖而造成的饥饿问题。线程编程实战书上推荐为默认的选择。
当然，newCachedThreadPool也会有缺点，当程序处理速度跟不上任务进入队列的速度时候会造成大量的线程，可能会耗完程序CPU。

这个时候newFixedThreadPool使我们另外一个选择，默认会创建一个corePoolSize为nThread，maximumPoolSize也为nThread，线程队列为LinkedBlockingQueue的线程池，`队列大小为Integer.maxValue，也就是说是一个无界队列`，这个很危险，当程序处理速度跟不上任务进入队列的速度时候会造成大量的线程，会有大量的对象积压在任务队列里面，直到耗完程序内存。
所以，如果我们的线程池有可能会面临高并发访问的场景，我们在创建的时候一定要小心，一定要用构造好合适的初始化值。

这里的合适的初始化值主要指，线程的corePoolSize，maximumPoolSize，workerQueue&handler，这里的workerQueue和handler尤为重要，`最好不要使用无界队列。使用有界队列最坏的情况可能会造成部分任务无法响应，但至少能保证大部分的任务正常执行。而且我们还可以定制自己的handler（饱和策略）来处理这些处理不了的任务。`

### 2.4线程池的饱和策略

当有界队列被填满了后，饱和策略开始发挥作用(`如果某个任务被提交到某一个已经被关闭的Executor时，也会用到饱和策略`)。ThreadPoolExecutor的饱和策略可以通过setRejectedExecutionHandler来修改。JDK提供几种不同的RejectedExecutionHandler实现，每种实现都包含不同的饱和策略：AbortPolicy，CallerRunsPolicy，DiscardPlolicy和DiscardOldestPolicy。
**"中止（Abort）"策略是默认的饱和策略，该策略将抛出未检查的RejectExecutionException**，调用者可以捕获这个异常，然后根据需求编写自己的处理代码。

当新提交的任务无法保存到该队列。“抛弃(Discard)”策略将会悄悄的抛弃该任务。

“抛弃最旧的（Discard-Oldest）”将会抛弃下一个将被执行的任务，然后尝试重新提交新的任务。**如果工作队列是一个优先队列，那么“抛弃最旧” 的饱和策略则是抛弃优先级最高的任务，因此最好不要将“抛弃最旧的”饱和策略和优先队列一起使用**。

“调用者运行（Caller-Runs）”实现了一种调节机制，该策略既不会抛弃任务，也不会抛出异常，而是将某些任务回退调用者，从而降低新任务流量。它不会在线程池的某个线程中执行新提交的任务，而是在一个调用了execute的线程中执行该任务。
我们可以将WebServer为例说明“调用者运行（Caller-Runs）”饱和策略。
当线程池中所有的线程都被占用，并且工作队列被填满后，下一个任务会调用executor时在主线程中执行。由于执行任务需要一定的时间，因此主线程至少在一段时间内不能提交任何任务，从而使得工作者线程有时间来处理正在执行的任务。在这期间主线程不会调用accept，因此到达的请求将被保存在TCP层的队列中而不是在应用程序的队列中。如果持续过载，那么TCP层将最终发现他的请求队列被填满，因此同样会开始抛弃请求。**当服务器过载时，这种过载情况会逐渐想外蔓延开来--从线程池到工作队列到应用程序到TCP层队列，最终到达客户端，导致服务器在高负载下实现一种平缓的性能降低。**

调用者运行饱和策略示例：

``` java
ThreadPoolExecutor executor = new ThreadPoolExecutor(N_THREADS, N_HREADS, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runable>(CAPCITY));
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRansPolicy());
```

### 2.5线程池的workQueue

线程池基本的任务排队方法有三种：无界队列，有界队列，和同步移交。

无界队列&有界队列对应的都是`LinkListBlockingQueue`，即是链式结构的阻塞队列，比较容易理解，暂不作详细探讨。

这里重点说明`SynchronousQueue`：基于BlockingQueue实现的，但实际上它不是一个真正的队列。因为他不会为队列中的元素维护存储空间，与其他的队列不同，它维护一组线程，这些线程在等待着把元素加入或者移除队列。这种队列的实现看似奇怪，但由于它可以直接交付工作，从而降低了数据从生产移动到消费者的延迟。直接交付的方式还可以将更多关于任务的状态反馈给生产者。

**注意，仅当有足够的消费者，并且总有一个消费者准备好获取交付的工作时，才适合使用同步队列。（只有当线程池是无界队列或者可以拒绝任务，同步队列才有使用价值）**
[newCacheThreadPool线程数量没有上线，所以newCacheThreadPool工厂方法使用了SynchronousQueue，]

### 2.6 线程池的关闭

线程池的关闭我们就要交给ExecutorService了，这也是为什么推荐使用Executors框架来替代线程（即便是单个的线程，也建议使用Executos框架）。

ExecutorService 提供了两个方法

``` java
/**
*正常关闭
*/
void shutdown();

/**
*首先关闭当前正在执行的任务，然后返回未启动的任务清单
**/
List<Runnable> shutdownNow();
```

两种关闭的差异来自安全性和响应性。强行关闭响应快，风险大。一般，没特殊要求，建议使用shutdown();

虽然调用shutdown程序会在关闭前执行完所有启动的线程，但是有的时候我们希望队列中所有的task都不要漏掉，不妨加一个个Jvm hook：

```java
Runtime.getRuntime().addShutdownHook(new Thread(){
   public void run(){
    try{
     //处理一些未完成的工作
    }catch (Exception e) {
     e.printStackTrace();
    }
   }
  });
```


## 3. 线程池的配置

### 3.1线程池大小

一般说来，大家认为线程池的大小经验值应该这样设置：（其中N为CPU的个数）

**如果是CPU密集型应用，则线程池大小设置为N+1**

**如果是IO密集型应用，则线程池大小设置为2N+1**

如果一台服务器上只部署这一个应用并且只有这一个线程池，那么这种估算或许合理，具体还需自行测试验证。
但是，IO优化中，这样的估算公式可能更适合：
最佳线程数目 = （（线程等待时间+线程CPU时间）/线程CPU时间 ）* CPU数目
因为很显然，线程等待时间所占比例越高，需要越多线程。线程CPU时间所占比例越高，需要越少线程。

下面举个例子：
比如平均每个线程CPU运行时间为0.5s，而线程等待时间（非CPU运行时间，比如IO）为1.5s，CPU核心数为8，那么根据上面这个公式估算得到：`((0.5+1.5)/0.5)*8=32`。这个公式进一步转化为：
**最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目**

这个只是估算值，实际通过测试调优得出合理值。

### 3.2 合理的配置线程池

要想合理的配置线程池，就必须首先分析任务特性，可以从以下几个角度来进行分析：

- **任务的性质**：CPU密集型任务，IO密集型任务和混合型任务。
- **任务的优先级**：高，中和低。
- **任务的执行时间**：长，中和短。
- **任务的依赖性**：是否依赖其他系统资源，如数据库连接。

任务性质不同的任务可以用不同规模的线程池分开处理。CPU密集型任务配置尽可能小的线程，如配置Ncpu+1个线程的线程池。IO密集型任务则由于线程并不是一直在执行任务，则配置尽可能多的线程，如2*Ncpu。混合型的任务，如果可以拆分，则将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐率要高于串行执行的吞吐率，如果这两个任务执行时间相差太大，则没必要进行分解。我们可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

`优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理`。它可以让优先级高的任务先得到执行，需要注意的是如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行。

`执行时间不同的任务可以交给不同规模的线程池来处理，或者也可以使用优先级队列，让执行时间短的任务先执行`。

依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，如果等待的时间越长CPU空闲时间就越长，那么线程数应该设置越大，这样才能更好的利用CPU。

建议使用有界队列，有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点，比如几千。有一次我们组使用的后台任务线程池的队列和线程池全满了，不断的抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞住，任务积压在线程池里。如果当时我们设置成无界队列，线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然我们的系统所有的任务是用的单独的服务器部署的，而我们使用不同规模的线程池跑不同类型的任务，但是出现这样问题时也会影响到其他任务。

## 4.线程池的应用与扩展

很多服务器都扩展的jdk自带的线程池来满足自己的需求，例如tomcat，motan...
通常，我们自己实现rpc服务器不可避免的用到线程池，参考借鉴他人的代码可以使我们少走很多弯路。

下面贴出motan中使用线程池的源代码供研究学习:

```java
	/*
	 *  Copyright 2009-2016 Weibo, Inc.
	 *
	 *    Licensed under the Apache License, Version 2.0 (the "License");
	 *    you may not use this file except in compliance with the License.
	 *    You may obtain a copy of the License at
	 *
	 *        http://www.apache.org/licenses/LICENSE-2.0
	 *
	 *    Unless required by applicable law or agreed to in writing, software
	 *    distributed under the License is distributed on an "AS IS" BASIS,
	 *    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
	 *    See the License for the specific language governing permissions and
	 *    limitations under the License.
	 */

	package com.weibo.api.motan.transport.netty;

	import java.util.concurrent.Executors;
	import java.util.concurrent.RejectedExecutionException;
	import java.util.concurrent.RejectedExecutionHandler;
	import java.util.concurrent.ThreadFactory;
	import java.util.concurrent.ThreadPoolExecutor;
	import java.util.concurrent.TimeUnit;
	import java.util.concurrent.atomic.AtomicInteger;

	import org.jboss.netty.util.internal.LinkedTransferQueue;

	/**
	 * <pre>
	 *
	 * 代码和思路主要来自于：
	 *
	 * tomcat :
	 * 		org.apache.catalina.core.StandardThreadExecutor
	 *
	 * java.util.concurrent
	 * threadPoolExecutor execute执行策略： 		优先offer到queue，queue满后再扩充线程到maxThread，如果已经到了maxThread就reject
	 * 						   		比较适合于CPU密集型应用（比如runnable内部执行的操作都在JVM内部，memory copy, or compute等等）
	 *
	 * StandardThreadExecutor execute执行策略：	优先扩充线程到maxThread，再offer到queue，如果满了就reject
	 * 						      	比较适合于业务处理需要远程资源的场景
	 *
	 * </pre>
	 *
	 * @author maijunsheng
	 * @version 创建时间：2013-6-20
	 *
	 */
	public class StandardThreadExecutor extends ThreadPoolExecutor {

	public static final int DEFAULT_MIN_THREADS = 20;
	public static final int DEFAULT_MAX_THREADS = 200;
	public static final int DEFAULT_MAX_IDLE_TIME = 60 * 1000; // 1 minutes

	protected AtomicInteger submittedTasksCount;	// 正在处理的任务数
	private int maxSubmittedTaskCount;				// 最大允许同时处理的任务数

	public StandardThreadExecutor() {
		this(DEFAULT_MIN_THREADS, DEFAULT_MAX_THREADS);
	}

	public StandardThreadExecutor(int coreThread, int maxThreads) {
		this(coreThread, maxThreads, maxThreads);
	}

	public StandardThreadExecutor(int coreThread, int maxThreads, long keepAliveTime, TimeUnit unit) {
		this(coreThread, maxThreads, keepAliveTime, unit, maxThreads);
	}

	public StandardThreadExecutor(int coreThreads, int maxThreads, int queueCapacity) {
		this(coreThreads, maxThreads, queueCapacity, Executors.defaultThreadFactory());
	}

	public StandardThreadExecutor(int coreThreads, int maxThreads, int queueCapacity, ThreadFactory threadFactory) {
		this(coreThreads, maxThreads, DEFAULT_MAX_IDLE_TIME, TimeUnit.MILLISECONDS, queueCapacity, threadFactory);
	}

	public StandardThreadExecutor(int coreThreads, int maxThreads, long keepAliveTime, TimeUnit unit, int queueCapacity) {
		this(coreThreads, maxThreads, keepAliveTime, unit, queueCapacity, Executors.defaultThreadFactory());
	}

	public StandardThreadExecutor(int coreThreads, int maxThreads, long keepAliveTime, TimeUnit unit,
			int queueCapacity, ThreadFactory threadFactory) {
		this(coreThreads, maxThreads, keepAliveTime, unit, queueCapacity, threadFactory, new AbortPolicy());
	}

	public StandardThreadExecutor(int coreThreads, int maxThreads, long keepAliveTime, TimeUnit unit,
			int queueCapacity, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
		super(coreThreads, maxThreads, keepAliveTime, unit, new ExecutorQueue(), threadFactory, handler);
		((ExecutorQueue) getQueue()).setStandardThreadExecutor(this);

		submittedTasksCount = new AtomicInteger(0);

		// 最大并发任务限制： 队列buffer数 + 最大线程数
		maxSubmittedTaskCount = queueCapacity + maxThreads;
	}

	public void execute(Runnable command) {
		int count = submittedTasksCount.incrementAndGet();

		// 超过最大的并发任务限制，进行 reject
		// 依赖的LinkedTransferQueue没有长度限制，因此这里进行控制
		if (count > maxSubmittedTaskCount) {
			submittedTasksCount.decrementAndGet();
			getRejectedExecutionHandler().rejectedExecution(command, this);
		}

		try {
			super.execute(command);
		} catch (RejectedExecutionException rx) {
			// there could have been contention around the queue
			if (!((ExecutorQueue) getQueue()).force(command)) {
				submittedTasksCount.decrementAndGet();

				getRejectedExecutionHandler().rejectedExecution(command, this);
			}
		}
	}

	public int getSubmittedTasksCount() {
		return this.submittedTasksCount.get();
	}

	public int getMaxSubmittedTaskCount() {
		return maxSubmittedTaskCount;
	}

	protected void afterExecute(Runnable r, Throwable t) {
		submittedTasksCount.decrementAndGet();
	}
}

/**
 * LinkedTransferQueue 能保证更高性能，相比与LinkedBlockingQueue有明显提升
 *
 * <pre>
 * 		1) 不过LinkedTransferQueue的缺点是没有队列长度控制，需要在外层协助控制
 * </pre>
 *
 * @author maijunsheng
 *
 */
class ExecutorQueue extends LinkedTransferQueue<Runnable> {
	private static final long serialVersionUID = -265236426751004839L;
	StandardThreadExecutor threadPoolExecutor;

	public ExecutorQueue() {
		super();
	}

	public void setStandardThreadExecutor(StandardThreadExecutor threadPoolExecutor) {
		this.threadPoolExecutor = threadPoolExecutor;
	}

	// 注：代码来源于 tomcat
	public boolean force(Runnable o) {
		if (threadPoolExecutor.isShutdown()) {
			throw new RejectedExecutionException("Executor not running, can't force a command into the queue");
		}
		// forces the item onto the queue, to be used if the task is rejected
		return super.offer(o);
	}

	// 注：tomcat的代码进行一些小变更
	public boolean offer(Runnable o) {
		int poolSize = threadPoolExecutor.getPoolSize();

		// we are maxed out on threads, simply queue the object
		if (poolSize == threadPoolExecutor.getMaximumPoolSize()) {
			return super.offer(o);
		}
		// we have idle threads, just add it to the queue
		// note that we don't use getActiveCount(), see BZ 49730
		if (threadPoolExecutor.getSubmittedTasksCount() <= poolSize) {
			return super.offer(o);
		}
		// if we have less threads than maximum force creation of a new
		// thread
		if (poolSize < threadPoolExecutor.getMaximumPoolSize()) {
			return false;
		}
		// if we reached here, we need to add it to the queue
		return super.offer(o);
	}
}

```

## 5.一些问题

### 5.1什么时候线程池能够最有效的提升性能？

>当大量任务相互独立且同构时才能体现出程序的工作负载分配到多个任务带来的真正性能提升。所以如果任务队列里面的任务有相互依赖，或者存在不同的任务，我们需要谨慎的处理。
>netty中的线程池就会同时处理IO任务和少量的定时任务，我们可以通过分配适当的执行比例ratio来优化。
>当线程中的任务出现依赖的情况，我们尤其要适当的增大线程池的大小，以防止任务阻塞，造成的线程饥饿。

### 5.2 如何处理线程池中部分运行时间较长的任务？

>如果线程池中出现运行时间较长的任务，即使不出现死锁，线程的响应速度也会变得非常糟糕。执行时间较长的任务不仅会造成线程池的阻塞，甚至还会增加执行时间较短的任务的服务时间。
>有一项技术可以缓存执行时间较长的任务的影响，即限定任务等待资源的时间，而不要无限制的等待。如果等待时间超时，那么可以把任务标志为失败，然后终止任务或者将任务重新放入队列。
>当然如果线程池中总是充满被阻塞的任务，那么也很有可能说明线程池的太小。

## 6.总结

使用线程池的时候我们要合理设置参数以满足我们的期望，比如corePoolSize&maxPoolSize&blokingQueueSize，`JDK默认的线程池会先初始化corePoolSize的线程数，如果不够，则继续offer满blokingQueueSize，直到达到blokingQueueSize才会增加maxPoolSize`。如果我们把blokingQueueSize设置无限大，显然maxPoolSize就没有作用了。对于一些IO密集的任务，我们需要大量的线程提升效率的时候显然是不合理的。

饱和策略一定也不要忽视，合理的饱和策略处理异常情况才能使我们的程序更加稳定。

线程池的大小一定要分情况讨论，首先考虑CPU密集或者IO密集，其次考虑任务是否有依赖关系，有依赖关系的需要稍稍设置大些，以免造成线程饥饿死锁。是否有长时间任务等等。

生产消费者模式是个好模式，有着广泛的应用。


**最后，再强调一遍，尽量避免使用无界队列！！！**

## 7.延伸阅读：

[江南白衣-Tomcat线程池，更符合大家想象的可扩展线程池](http://calvin1978.blogcn.com/articles/tomcat-threadpool.html)

[聊聊并发（三）——JAVA线程池的分析和使用](http://www.infoq.com/cn/articles/java-threadPool)

---

写在后面：

[注]：未经许可，严禁转载[文章会随时更新补充...随时修改可能错误的观点]

写一遍文章真累...org





