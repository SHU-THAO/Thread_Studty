# Java 并发：深入理解 ThreadLocal

​	ThreadLocal 又名线程局部变量，是 Java 中一种较为特殊的线程绑定机制，用于保证变量在不同线程间的隔离性，以方便每个线程处理自己的状态。进一步地，本文以ThreadLocal类的源码为切入点，深入分析了ThreadLocal类的作用原理，并给出应用场景和一般使用步骤。

## 对 ThreadLocal 的理解

### 1). ThreadLocal 概述

​	ThreadLocal 又名 **线程局部变量**  **，是 Java 中一种较为特殊的线程绑定机制，可以为每一个使用该变量的线程都提供一个变量值的副本，并且每一个线程都可以独立地改变自己的副本，而不会与其它线程的副本发生冲突。**一般而言，通过 ThreadLocal 存取的数据总是与当前线程相关，也就是说，JVM 为每个运行的线程绑定了私有的本地实例存取空间，从而为多线程环境常出现的并发访问问题提供了一种 **隔离机制** 。

​	如果一段代码中所需要的**数据必须与其他代码**共享，那就看看这些共享数据的代码能否保证在同一个线程中执行？如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。也就是说，**如果一个某个变量要被某个线程 独享，那么我们就可以通过ThreadLocal来实现线程本地存储功能。**

​	

### 2). ThreadLocal 在 JDK 中的定义

![1557905702500](C:\Users\12084\AppData\Roaming\Typora\typora-user-images\1557905702500.png)

``` java
public class Thread implements Runnable {

    ...

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    ...
}
```

我们可以从中摘出三条要点：

- **每个线程都有关于该 ThreadLocal变量 的私有值** 
  每个线程都有一个独立于其他线程的上下文来保存这个变量的值，并且对其他线程是不可见的。
- **独立于变量的初始值** 
  ThreadLocal 可以给定一个初始值，这样每个线程就会获得这个初始化值的一个拷贝，并且每个线程对这个值的修改对其他线程是不可见的。
- **将某个类的状态与线程相关联** 
  我们从JDK中对ThreadLocal的描述中可以看出，ThreadLocal的一个重要作用是就是将类的状态与线程关联起来，这个时候通常的解决方案就是在这个类中定义一个 private static ThreadLocal 实例。



### 3). 应用场景

​	**类 ThreadLocal 主要解决的就是为每个线程绑定自己的值，以方便其处理自己的状态。**形象地讲，可以将 ThreadLocal变量 比喻成全局存放数据的盒子，盒子中可以存储每个线程的私有数据。例如，以下类用于生成对每个线程唯一的局部标识符。线程 ID 是在第一次调用 uniqueNum.get() 时分配的，在后续调用中不会更改。

``` java
import java.util.concurrent.atomic.AtomicInteger;

public class UniqueThreadIdGenerator {
    private static final AtomicInteger uniqueId = new AtomicInteger(0);

    private static final ThreadLocal<Integer> uniqueNum = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return uniqueId.getAndIncrement();
        }
    };

    public static void main(String[] args) {
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            String name = "Thread-" + i;
            threads[i] = new Thread(name){
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName() + ": "
                            + uniqueNum.get());
                }
            };
            threads[i].start();
        }

        System.out.println(Thread.currentThread().getName() + ": "
                + uniqueNum.get());
    }
}
/* Output(输出结果不唯一): 
        Thread-1: 2
        Thread-0: 0
        Thread-2: 3
        main: 1
        Thread-3: 4
        Thread-4: 5
 *///:~
```



## 深入分析ThreadLocal类

​	下面，我们来看一下 ThreadLocal 的具体实现，该类一共提供的四个方法：

``` java
	public T get() { }
	public void set(T value) { }
	public void remove() { }
	protected T initialValue() { }
```

​	其中，get()方法是用来获取 ThreadLocal变量 在当前线程中保存的值，set() 用来设置 ThreadLocal变量 在当前线程中的值，remove() 用来移除当前线程中相关 ThreadLocal变量，initialValue() 是一个 protected 方法，一般需要重写。

### **1、 原理探究**

#### 1). 切入点：get()

首先，我们先看其源码：

``` java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();    // 获取当前线程对象
        ThreadLocalMap map = getMap(t);     // 获取当前线程的成员变量 threadLocals
        if (map != null) {
            // 从当前线程的 ThreadLocalMap 获取该 thread-local variable 对应的 entry
            ThreadLocalMap.Entry e = map.getEntry(this);    
            if (e != null)      
                return (T)e.value;   // 取得目标值
        }
        return setInitialValue();  
    }
```

#### 2). 关键点：setInitialValue()

``` java
/**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();     // 默认实现返回 null
        Thread t = Thread.currentThread();   // 获得当前线程
        ThreadLocalMap map = getMap(t);     // 得到当前线程 ThreadLocalMap类型域 threadLocals
        if (map != null)
            map.set(this, value);  // 该 map 的键是当前 ThreadLocal 对象
        else
            createMap(t, value);   
        return value;
    }
```

​	我们紧接着看上述方法涉及到的三个方法：initialValue()，set(this, value) 和 createMap(t, value)。

**(1) initialValue()**

``` java
   /**
     * Returns the current thread's "initial value" for this
     * thread-local variable.  This method will be invoked the first
     * time a thread accesses the variable with the {@link #get}
     * method, unless the thread previously invoked the {@link #set}
     * method, in which case the <tt>initialValue</tt> method will not
     * be invoked for the thread.  Normally, this method is invoked at
     * most once per thread, but it may be invoked again in case of
     * subsequent invocations of {@link #remove} followed by {@link #get}.
     *
     * <p>This implementation simply returns <tt>null</tt>; if the
     * programmer desires thread-local variables to have an initial
     * value other than <tt>null</tt>, <tt>ThreadLocal</tt> must be
     * subclassed, and this method overridden.  Typically, an
     * anonymous inner class will be used.
     *
     * @return the initial value for this thread-local
     */
    protected T initialValue() {
        return null;            // 默认实现返回 null
    }
```

**(2) createMap()**

``` java
/**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     * @param map the map to store.
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue); // this 指代当前 ThreadLocal 变量，为 map 的键  
    }
```



至此，可能大部分朋友已经明白了 ThreadLocal类 是如何为每个线程创建变量的副本的：

- **每个线程内部有一个 ThreadLocal.ThreadLocalMap 类型的成员变量 threadLocals，这个 threadLocals 存储了与该线程相关的所有 ThreadLocal 变量及其对应的值（”ThreadLocal 变量及其对应的值” 就是该Map中的一个 Entry）。**我们知道，Map 中存放的是一个个 Entry，其中每个 Entry 都包含一个 Key 和一个 Value。在这里，就threadLocals 而言，它的 Entry 的 Key 是 ThreadLocal 变量， Value 是该 ThreadLocal 变量对应的值；
- **初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的值为value，存到 threadLocals；**
- **然后在当前线程里面，如果要使用ThreadLocal对象，就可以通过get方法获得该线程的threadLocals，然后以该ThreadLocal对象为键取得其对应的 Value，也就是ThreadLocal对象中所存储的值。**



### 2、实例验证

​	下面通过一个例子来证明通过ThreadLocal能达到在每个线程中创建变量副本的效果：

``` java
public class Test {

    ThreadLocal<Long> longLocal = new ThreadLocal<Long>();
    ThreadLocal<String> stringLocal = new ThreadLocal<String>();

    public void set() {
        longLocal.set(Thread.currentThread().getId());
        stringLocal.set(Thread.currentThread().getName());
    }

    public long getLong() {
        return longLocal.get();
    }

    public String getString() {
        return stringLocal.get();
    }

    public static void main(String[] args) throws InterruptedException {
        final Test test = new Test();

        test.set();
        System.out.println("父线程 main ：");
        System.out.println(test.getLong());
        System.out.println(test.getString());

        Thread thread1 = new Thread() {
            public void run() {
                test.set();
                System.out.println("\n子线程 Thread-0 ：");
                System.out.println(test.getLong());
                System.out.println(test.getString());
            };
        };
        thread1.start();
    }
}
/* Output: 
        父线程 main ：
                    1
                    main

        子线程 Thread-0 ：
                    12
                    Thread-0
 *///:~
```

​	从这段代码的输出结果可以看出，在main线程中和thread1线程中，longLocal保存的副本值和stringLocal保存的副本值都不一样，并且进一步得出：

- 实际上，通过 ThreadLocal 创建的副本是存储在每个线程自己的threadLocals中的；
- 为何 threadLocals 的类型 ThreadLocalMap 的键值为 ThreadLocal 对象，因为每个线程中可有多个 threadLocal变量，就像上面代码中的 longLocal 和 stringLocal；
- 在进行get之前，必须先set，否则会报空指针异常；若想在get之前不需要调用set就能正常访问的话，必须重写initialValue()方法。



## ThreadLocal的应用场景

​	在 Java 中，类 ThreadLocal 解决的是变量在不同线程间的隔离性。最常见的 ThreadLocal 使用场景有 数据库连接问题、Session管理等。

### (1) 数据库连接问题

``` java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};

public static Connection getConnection() {
    return connectionHolder.get();
}
```

### (2) Session管理

``` java
private static final ThreadLocal threadSession = new ThreadLocal();

public static Session getSession() throws InfrastructureException {
    Session s = (Session) threadSession.get();
    try {
        if (s == null) {
            s = getSessionFactory().openSession();
            threadSession.set(s);
        }
    } catch (HibernateException ex) {
        throw new InfrastructureException(ex);
    }
    return s;
}
```

### (3) Thread-per-Request (一个请求对应一个服务器线程)

​	在经典Web交互模型中，请求的处理基本上采用的都是“一个请求对应一个服务器线程”的处理方式，因此就可以将请求设置成类似ThreadLocal<Request>的形式，这样，当某个服务器线程来处理请求时，就可以独享该请求的处理了。



##  ThreadLocal 一般使用步骤

ThreadLocal 使用步骤一般分为三步：

- 创建一个 ThreadLocal 对象 threadXxx，用来保存线程间需要隔离处理的对象 xxx；
- 提供一个获取要隔离访问的数据的方法 getXxx()，在方法中判断，若 ThreadLocal对象为null时候，应该 new() 一个隔离访问类型的对象；
- 在线程类的run()方法中，通过getXxx()方法获取要操作的数据，这样可以保证每个线程对应一个数据对象，在任何时刻都操作的是这个对象，不会交叉。