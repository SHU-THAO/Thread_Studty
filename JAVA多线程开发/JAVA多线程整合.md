# 面试/笔试第四弹 —— 多线程面试问题集锦

### **1、如何停止一个线程**

- 使用volatile变量终止正常运行的线程 + 抛异常法/Return法
- 组合使用interrupt方法与interruptted/isinterrupted方法终止正在运行的线程 + 抛异常法/Return法
- 使用interrupt方法终止 正在阻塞中的 线程



### **2、何为线程安全的类？**

​	在线程安全性的定义中，最核心的概念就是 正确性。当多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类都能表现出正确的行为，那么这个类就是线程安全的。



### **3、为什么线程通信的方法wait(), notify()和notifyAll()被定义在Object类里？**

``` java
Object lock = new Object();
synchronized (lock) {
    lock.wait();
    ...
}
```

​	Wait-notify机制是在获取对象锁的前提下不同线程间的通信机制。在Java中，任意对象都可以当作锁来使用，由于锁对象的任意性，所以这些通信方法需要被定义在Object类里。



### **4、为什么wait(), notify()和notifyAll()必须在同步方法或者同步块中被调用？**

​	wait/notify机制是依赖于Java中Synchronized同步机制的，其目的在于确保等待线程从Wait()返回时能够感知通知线程对共享变量所作出的修改。如果不在同步范围内使用，就会抛出java.lang.IllegalMonitorStateException的异常。



### **5、并发三准则**

- 异常不会导致死锁现象：当线程出现异常且没有捕获处理时，JVM会自动释放当前线程占用的锁，因此不会由于异常导致出现死锁现象，同时还会释放CPU；
- 锁的是对象而非引用；
- 有wait必有notify；



### **6、如何确保线程安全？**

在Java中可以有很多方法来保证线程安全，诸如：

- 通过加锁(Lock/Synchronized)保证对临界资源的同步互斥访问；
- 使用volatile关键字，轻量级同步机制，但不保证原子性；（可见性和有序性）
- 使用不变类 和 线程安全类(原子类，并发容器，同步容器等)。（String+ConcurrentHashMap）



### **7、volatile关键字在Java中有什么作用**

​	volatile的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新，即保证了内存的可见性，除此之外还能 禁止指令重排序。此外，synchronized关键字也可以保证内存可见性。

​	指令重排序问题在并发环境下会导致线程安全问题，volatile关键字通过禁止指令重排序来避免这一问题。而对于Synchronized关键字，其所控制范围内的程序在执行时独占的，指令重排序问题不会对其产生任何影响，因此无论如何，其都可以保证最终的正确性。



### **8、ThreadLocal及其引发的内存泄露**

​	ThreadLocal是Java中的一种线程绑定机制，可以为每一个使用该变量的线程都提供一个变量值的副本，并且每一个线程都可以独立地改变自己的副本，而不会与其它线程的副本发生冲突。

​	每个线程内部有一个 ThreadLocal.ThreadLocalMap 类型的成员变量 threadLocals，这个 threadLocals 存储了与该线程相关的所有 ThreadLocal 变量及其对应的值，也就是说，ThreadLocal 变量及其对应的值就是该Map中的一个 Entry，更直白地，threadLocals中每个Entry的Key是ThreadLocal 变量本身，而Value是该ThreadLocal变量对应的值。



#### ThreadLocal可能引起的内存泄露

​	　下面是ThreadLocalMap的部分源码，我们可以看出ThreadLocalMap里面对Key的引用是弱引用。那么，就存在这样的情况：当释放掉对threadlocal对象的强引用后，map里面的value没有被回收，但却永远不会被访问到了，因此ThreadLocal存在着内存泄露问题。

``` java
    static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }
        ...
    }
```

​	看下面的图示， 实线代表强引用，虚线代表弱引用。每个thread中都存在一个map，map的类型是上文提到的ThreadLocal.ThreadLocalMap，该map中的key为一个ThreadLocal实例。这个Map的确使用了弱引用，不过弱引用只是针对key，每个key都弱引用指向ThreadLocal对象。一旦把threadlocal实例置为null以后，那么将没有任何强引用指向ThreadLocal对象，因此ThreadLocal对象将会被 Java GC 回收。但是，与之关联的value却不能回收，因为存在一条从current thread连接过来的强引用。 只有当前thread结束以后，current thread就不会存在栈中，强引用断开，Current Thread、Map及value将全部被Java GC回收。

​	所以，得出一个结论就是：只要这个线程对象被Java GC回收，就不会出现内存泄露。但是如果只把ThreadLocal引用指向null而线程对象依然存在，那么此时Value是不会被回收的，这就发生了我们认为的内存泄露。比如，在使用线程池的时候，线程结束是不会销毁的而是会再次使用的，这种情形下就可能出现ThreadLocal内存泄露。　　

​	Java为了最小化减少内存泄露的可能性和影响，在ThreadLocal进行get、set操作时会清除线程Map里所有key为null的value。所以最怕的情况就是，ThreadLocal对象设null了，开始发生“内存泄露”，然后使用线程池，线程结束后被放回线程池中而不销毁，那么如果这个线程一直不被使用或者分配使用了又不再调用get/set方法，那么这个期间就会发生真正的内存泄露。因此，最好的做法是：**在不使用该ThreadLocal对象时，及时调用该对象的remove方法去移除ThreadLocal.ThreadLocalMap中的对应Entry。**



### **9、什么是死锁(Deadlock)？如何分析和避免死锁？**

​	死锁是指两个以上的线程永远阻塞的情况，这种情况产生至少需要两个以上的线程和两个以上的资源。

​	分析死锁，我们需要查看Java应用程序的线程转储。我们需要找出那些状态为BLOCKED的线程和他们等待的资源。每个资源都有一个唯一的id，用这个id我们可以找出哪些线程已经拥有了它的对象锁。下面列举了一些JDK自带的死锁检测工具：

**(1). Jconsole：**JDK自带的图形化界面工具，主要用于对 Java 应用程序做性能分析和调优。

![Jconsoleæ­"é.png-26.7kB](C:\Users\12084\Desktop\JAVA多线程开发\Jconsole死锁检测.png)

**(2). Jstack：**JDK自带的命令行工具，主要用于线程Dump分析。

![è¿éåå¾çæè¿°](C:\Users\12084\Desktop\JAVA多线程开发\Jstack.png)

**(3). VisualVM：**JDK自带的图形化界面工具，主要用于对 Java 应用程序做性能分析和调优。

![VisualVMæ­"é.jpg-240.8kB](C:\Users\12084\Desktop\JAVA多线程开发\VisualVM图形化界面.jpg)



### **10、什么是Java Timer类？如何创建一个有特定时间间隔的任务？**

​	Timer是一个调度器，可以用于安排一个任务在未来的某个特定时间执行或周期性执行。TimerTask是一个实现了Runnable接口的抽象类，我们需要去继承这个类来创建我们自己的定时任务并使用Timer去安排它的执行。

``` java
	Timer timer = new Timer();
	timer.schedule(new TimerTask() {
        public void run() {
            System.out.println("abc");
        }
	}, 200000 , 1000);
```



### **11、什么是线程池？如何创建一个Java线程池？**

​	一个线程池管理了一组工作线程，同时它还包括了一个用于放置等待执行的任务的队列。线程池可以避免线程的频繁创建与销毁，降低资源的消耗，提高系统的反应速度。java.util.concurrent.Executors提供了几个java.util.concurrent.Executor接口的实现用于创建线程池，其主要涉及四个角色：

- 线程池：Executor
- 工作线程：Worker线程，Worker的run()方法执行Job的run()方法
- 任务Job：Runable和Callable
- 阻塞队列：BlockingQueue

![çº¿ç¨æ± çå¤çæµç¨.jpg-25.3kB](C:\Users\12084\Desktop\JAVA多线程开发\线程池.jpg)

#### 线程池Executor

​	在一个应用程序中，我们需要多次使用线程，也就意味着，我们需要多次创建并销毁线程。而创建并销毁线程的过程势必会消耗内存。而在Java中，内存资源是及其宝贵的，所以，我们就提出了线程池的概念。

​        Executor及其实现类是用户级的线程调度器，也是对任务执行机制的抽象，其将任务的提交与任务的执行分离开来，核心实现类包括ThreadPoolExecutor(用来执行被提交的任务)和ScheduledThreadPoolExecutor(可以在给定的延迟后执行任务或者周期性执行任务)。

​	而我们创建时，一般使用它的子类：ThreadPoolExecutor.

``` java
public ThreadPoolExecutor(int corePoolSize,  
                              int maximumPoolSize,  
                              long keepAliveTime,  
                              TimeUnit unit,  
                              BlockingQueue<Runnable> workQueue,  
                              ThreadFactory threadFactory,  
                              RejectedExecutionHandler handler)
}{}
```

​	这是其中最重要的一个构造方法，这个方法决定了创建出来的线程池的各种属性，下面依靠一张图来更好的理解线程池和这几个参数：

![img](C:\Users\12084\Desktop\JAVA多线程开发\线程池参数.png)

​	可以看出，线程池中的corePoolSize就是线程池中的核心线程数量，这几个核心线程，即使在没有用的时候，也不会被回收，maximumPoolSize就是线程池中可以容纳的最大线程的数量，而keepAliveTime，就是线程池中除了核心线程之外的其他的最长可以保留的时间，因为在线程池中，除了核心线程即使在无任务的情况下也不能被清除，其余的都是有存活时间的，意思就是非核心线程可以保留的最长的空闲时间，而util，就是计算这个时间的一个单位，workQueue，就是等待队列，任务可以储存在任务队列中等待被执行，执行的是FIFIO原则（先进先出）。threadFactory，就是创建线程的线程工厂，最后一个handler,是一种拒绝策略，我们可以在任务满了时执行，或者拒绝执行某些任务。



#### 任务Runable/Callable

​	Runnable(run)和Callable(call)都是对任务的抽象，但是Callable可以返回任务执行的结果或者抛出异常。



#### 任务执行状态Future

​	Future是对任务执行状态和结果的抽象，核心实现类是furtureTask (所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值) ；

![FutureAPI.png-9.4kB](C:\Users\12084\Desktop\JAVA多线程开发\FutureAPI.png)

-  使用Callable+Future获取执行结果

``` java
    ExecutorService executor = Executors.newCachedThreadPool();
    Task task = new Task();
    Future<Integer> result = executor.submit(task);
    System.out.println("task运行结果" + result.get());    

    class Task implements Callable<Integer>{
            @Override
            public Integer call() throws Exception {
                System.out.println("子线程在进行计算");
                Thread.sleep(3000);
                int sum = 0;
                for(int i=0;i<100;i++)
                    sum += i;
                return sum;
     		}
    }
```

- 使用Callable + FutureTask获取执行结果

``` java
    ExecutorService executor = Executors.newCachedThreadPool();
     Task task = new Task();
     FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
     executor.submit(futureTask);  
    System.out.println("task运行结果"+futureTask.get());

    class Task implements Callable<Integer>{
            @Override
            public Integer call() throws Exception {
                System.out.println("子线程在进行计算");
                Thread.sleep(3000);
                int sum = 0;
                for(int i=0;i<100;i++)
                    sum += i;
                return sum;
     		}
    }
```



#### 四种常用的线程池

- FixedThreadPool

用于创建使用固定线程数的ThreadPool，corePoolSize = maximumPoolSize = n(固定的含义)，阻塞队列为LinkedBlockingQueue。

``` java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

- SingleThreadExecutor

用于创建一个单线程的线程池，corePoolSize = maximumPoolSize = 1，阻塞队列为LinkedBlockingQueue。

``` java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

- CachedThreadPool

用于创建一个可缓存的线程池，corePoolSize = 0， maximumPoolSize = Integer.MAX_VALUE，阻塞队列为SynchronousQueue(没有容量的阻塞队列，每个插入操作必须等待另一个线程对应的移除操作，反之亦然)。

``` java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

- ScheduledThreadPoolExecutor

用于创建一个大小无限的线程池，此线程池支持定时以及周期性执行任务的需求。

``` java
	public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
              new DelayedWorkQueue());
    }
```



#### 线程池的饱和策略

​	当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略：

- AbortPolicy：直接抛出异常，默认策略；
- CallerRunsPolicy：用调用者所在的线程来执行任务；
- DiscardOldestPolicy：丢弃阻塞队列中最老的任务，并执行当前任务；
- DiscardPolicy：直接丢弃任务；

​        当然也可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。



#### 线程池调优

- 设置最大线程数，防止线程资源耗尽；
- 使用有界队列，从而增加系统的稳定性和预警能力(饱和策略)；
- 根据任务的性质设置线程池大小：CPU密集型任务(CPU个数个线程)，IO密集型任务(CPU个数两倍的线程)，混合型任务(拆分)。



### **12、CAS ： CAS自旋volatile变量，是一种很经典的用法。**

​	CAS，Compare and Swap即比较并交换，设计并发算法时常用到的一种技术。CAS有3个操作数，内存值V，旧的预期值A，新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。CAS是通过unsafe类的compareAndSwap (JNI, Java Native Interface) 方法实现的，该方法包括四个参数：第一个参数是要修改的对象，第二个参数是对象中要修改变量的偏移量，第三个参数是修改之前的值，第四个参数是预想修改后的值。

​	CAS虽然很高效的解决原子操作，但是CAS仍然存在三大问题：**ABA问题、循环时间长开销大和只能保证一个共享变量的原子操作。**

- **ABA问题：**因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。
- **不适用于竞争激烈的情形中：**并发越高，失败的次数会越多，CAS如果长时间不成功，会极大的增加CPU的开销。因此CAS不适合竞争十分频繁的场景。
- **只能保证一个共享变量的原子操作：**当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，因此可以把多个变量放在一个对象里来进行CAS操作。



### **13、AQS ： 队列同步器**

​	队列同步器(AbstractQueuedSynchronizer)是用来构建锁和其他同步组件的基础框架，技术是 **CAS自旋Volatile变量**：它使用了一个Volatile成员变量表示同步状态，通过CAS修改该变量的值，修改成功的线程表示获取到该锁；若没有修改成功，或者发现状态state已经是加锁状态，则通过一个Waiter对象封装线程，添加到等待队列中，并挂起等待被唤醒。

​	同步器是实现锁的关键，子类通过继承同步器并实现它的抽象方法来管理同步状态，利用同步器实现锁的语义。特别地，锁是面向锁使用者的，它定义了使用者与锁交互的接口，隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程排队、等待与唤醒等底层操作。**锁和同步器很好地隔离了锁的使用者与锁的实现者所需关注的领域。**

​	一般来说，自定义同步器要么是独占方式，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

​	同步器的设计是基于 **模板方法模式** 的，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

![AQS.png-26.4kB](C:\Users\12084\Desktop\JAVA多线程开发\AQS.png)

​	AQS维护了一个volatile int state（代表共享资源）和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）。这里volatile是核心关键词，具体volatile的语义，在此不述。state的访问方式有三种:getState()、setState()以及compareAndSetState()。

​	AQS定义了两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它；
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false；
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false；
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源；
- tryReleaseShared(int)：共享方式。尝试释放资源，成功则返回true，失败则返回false。

​         以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。



### **14、Java Concurrency API中的Lock接口(Lock interface)是什么？对比同步它有什么优势？**

​	synchronized是Java的关键字，是Java的内置特性，在JVM层面实现了对临界资源的同步互斥访问。Synchronized的语义底层是通过一个monitor对象来完成的，线程执行monitorenter/monitorexit指令完成锁的获取与释放。而Lock是一个Java接口(API如下图所示)，是基于JDK层面实现的，通过这个接口可以实现同步访问，它提供了比synchronized关键字更灵活、更广泛、粒度更细的锁操作，底层是由AQS实现的。二者之间的差异总结如下：

- 实现层面：synchronized（JVM层面）、Lock（JDK层面）
- 响应中断：Lock 可以让等待锁的线程响应中断，而使用synchronized时，等待的线程会一直等待下去，不能够响应中断；
- 立即返回：可以让线程尝试获取锁，并在无法获取锁的时候立即返回或者等待一段时间，而synchronized却无法办到；
- 读写锁：Lock可以提高多个线程进行读操作的效率
- 可实现公平锁：Lock可以实现公平锁，而sychronized天生就是非公平锁
- 显式获取和释放：synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；

![LOCK API.png-36.5kB](C:\Users\12084\Desktop\JAVA多线程开发\LOCKAPI.png)



### **15、Condition**

Condition可以用来实现线程的分组通信与协作。以生产者/消费者问题为例，

- wait/notify/notifyAll:在队列为空时，通知所有线程；在队列满时，通知所有线程，防止生产者通知生产者，消费者通知消费者的情形产生。
- await/signal/signalAll：将线程分为消费者线程和生产者线程两组：在队列为空时，通知生产者线程生产；在队列满时，通知消费者线程消费。

![](C:\Users\12084\Desktop\JAVA多线程开发\Condition.png)



### **16、什么是阻塞队列？如何使用阻塞队列来实现生产者-消费者模型？**

​	java.util.concurrent.BlockingQueue的特性是：当队列是空的时，从队列中获取或删除元素的操作将会被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。特别地，阻塞队列不接受空值，当你尝试向队列中添加空值的时候，它会抛出NullPointerException。另外，阻塞队列的实现都是线程安全的，所有的查询方法都是原子的并且使用了内部锁或者其他形式的并发控制。

​	BlockingQueue 接口是java collections框架的一部分，它主要用于实现生产者-消费者问题。特别地，SynchronousQueue是一个没有容量的阻塞队列，每个插入操作必须等待另一个线程的对应移除操作，反之亦然。CachedThreadPool使用SynchronousQueue把主线程提交的任务传递给空闲线程执行。



### **17、同步容器（强一致性）**

​	同步容器指的是 Vector、Stack、HashTable及Collections类中提供的静态工厂方法创建的类。其中，Vector实现了List接口，Vector实际上就是一个数组，和ArrayList类似，但是Vector中的方法都是synchronized方法，即进行了同步措施；Stack也是一个同步容器，它的方法也用synchronized进行了同步，它实际上是继承于Vector类；HashTable实现了Map接口，它和HashMap很相似，但是HashTable进行了同步处理，而HashMap没有。

​	Collections类是一个工具提供类，注意，它和Collection不同，Collection是一个顶层的接口。在Collections类中提供了大量的方法，比如对集合或者容器进行排序、查找等操作。最重要的是，在它里面提供了几个静态工厂方法来创建同步容器类，如下图所示：

![åæ­¥å®¹å¨.png-145.7kB](C:\Users\12084\Desktop\JAVA多线程开发\Collections.png)

### **18、什么是CopyOnWrite容器(弱一致性)？**

​	CopyOnWrite容器即写时复制的容器，适用于读操作远多于修改操作的并发场景中。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。

​	从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器，它们是CopyOnWriteArrayList和CopyOnWriteArraySet。CopyOnWrite容器主要存在两个弱点：

- 容器对象的复制需要一定的开销，如果对象占用内存过大，可能造成频繁的YoungGC和Full GC；
- CopyOnWriteArrayList不能保证数据实时一致性，只能保证最终一致性。



### **19、ConcurrentHashMap (弱一致性)**

​	**ConcurrentHashMap的弱一致性主要是为了提升效率，也是一致性与效率之间的一种权衡。**要成为强一致性，就得到处使用锁，甚至是全局锁，这就与Hashtable和同步的HashMap一样了。ConcurrentHashMap的弱一致性主要体现在以下几方面：

- get操作是弱一致的：get操作只能保证一定能看到已完成的put操作；

![ConcurrentHashMapçå¼±ä¸è´æ§.png-136.8kB](C:\Users\12084\Desktop\JAVA多线程开发\ConcurrentHashMap_get.png)

- clear操作是弱一致的：在清除完一个segments之后，正在清理下一个segments的时候，已经清理的segments可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。

``` java
public void clear() {
    for (int i = 0; i < segments.length; ++i)
        segments[i].clear();
}
```

- ConcurrentHashMap中的迭代操作是弱一致的(未遍历的内容发生变化可能会反映出来)：在遍历过程中，如果已经遍历的数组上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中。



### **20、happens-before**

​	happens-before 指定了两个操作间的执行顺序：如果 A happens before B，那么Java内存模型将向程序员保证 —— A 的执行顺序排在 B 之前，并且 A 操作的结果将对 B 可见，其具体包括如下8条规则：

- 程序顺序规则：单线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作；
- 管程锁定规则：一个unlock操作先行发生于对同一个锁的lock操作；
- volatile变量规则：对一个Volatile变量的写操作先行发生于对这个变量的读操作；
- 线程启动规则：Thread对象的start()方法先行发生于此线程的其他动作；
- 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
- 线程终止规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行；
- 对象终结规则：一个对象的初始化完成先行发生于它的finalize()方法的开始；
- 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；



### **21、锁优化技术**

​	锁优化技术的目的在于线程之间更高效的共享数据，解决竞争问题，更好提高程序执行效率。

- **自旋锁(上下文切换代价大)：**互斥锁 -> 阻塞 –> 释放CPU，线程上下文切换代价较大 + 共享变量的锁定时间较短 == 让线程通过自旋等一会儿，自旋锁
- **锁粗化(一个大锁优于若干小锁)：**一系列连续操作对同一对象的反复频繁加锁/解锁会导致不必要的性能损耗，建议粗化锁 
  一般而言，同步范围越小越好，这样便于其他线程尽快拿到锁，但仍然存在特例。
- **偏向锁(有锁但当前情形不存在竞争)：**消除数据在无竞争情况下的同步原语，提高带有同步但无竞争的程序性能。
- **锁消除(有锁但不存在竞争，锁多余)：**JVM编译优化，将不存在数据竞争的锁消除



### **22、主线程等待子线程运行完毕再运行的方法**

#### join

​	Thread提供了让一个线程等待另一个线程完成的方法 — join()方法。当在某个程序执行流程中调用其它线程的join()方法时，调用线程将被阻塞，直到被join()方法加入的join线程执行完毕为止，在继续运行。join()方法的实现原理是不停检查join线程是否存活，如果join线程存活则让当前线程永远等待。直到join线程完成后，线程的this.notifyAll()方法会被调用。

####  CountDownLatch

​	Countdown Latch允许一个或多个线程等待其他线程完成操作。CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完成，这里就传入N。当我们调用countDown方法时，N就会减1，await方法会阻塞当前线程，直到N变成0。这里说的N个点，可以使用N个线程，也可以是1个线程里的N个执行步骤。

![CountDownLatchçä½¿ç¨.png-53.9kB](C:\Users\12084\Desktop\JAVA多线程开发\CountDownLatch.png)

#### Sleep

​	用sleep方法，让主线程睡眠一段时间，当然这个睡眠时间是主观的时间，是我们自己定的，这个方法不推荐，但是在这里还是写一下，毕竟是解决方法。