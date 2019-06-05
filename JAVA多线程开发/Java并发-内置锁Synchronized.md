# Java 并发：内置锁 Synchronized

**摘要：** 

​	在多线程编程中，线程安全问题是一个最为关键的问题，其核心概念就在于正确性，即当多个线程访问某一共享、可变数据时，始终都不会导致数据破坏以及其他不该出现的结果。而所有的并发模式在解决这个问题时，采用的方案都是序列化访问临界资源 。在 Java 中，提供了两种方式来实现同步互斥访问：synchronized 和 Lock。本文针对 synchronized 内置锁 详细讨论了其在 Java 并发 中的应用，包括它的具体使用场景（同步方法、同步代码块、实例对象锁 和 Class 对象锁）、可重入性 和 注意事项。



## 线程安全问题

​	在单线程中不会出现线程安全问题，而在多线程编程中，有可能会出现同时访问同一个 **共享、可变资源** 的情况，这种资源可以是：一个变量、一个对象、一个文件等。特别注意两点：

- **共享： 意味着该资源可以由多个线程同时访问；**
- **可变： 意味着该资源可以在其生命周期内被修改。**

​        所以，当多个线程同时访问这种资源的时候，就会存在一个问题：

　　**由于每个线程执行的过程是不可控的，所以需要采用同步机制来协同对对象可变状态的访问。** 



举个 **数据脏读** 的例子：

``` java
//资源类
class PublicVar {

    public String username = "A";
    public String password = "AA";

    //同步实例方法
    public synchronized void setValue(String username, String password) {
        try {
            this.username = username;
            Thread.sleep(5000);
            this.password = password;

            System.out.println("method=setValue " +"\t" + "threadName="
                    + Thread.currentThread().getName() + "\t" + "username="
                    + username + ", password=" + password);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    //非同步实例方法
    public void getValue() {
        System.out.println("method=getValue " + "\t" +  "threadName="
                + Thread.currentThread().getName()+ "\t" + " username=" + username
                + ", password=" + password);
    }
}


//线程类
class ThreadA extends Thread {

    private PublicVar publicVar;

    public ThreadA(PublicVar publicVar) {
        super();
        this.publicVar = publicVar;
    }

    @Override
    public void run() {
        super.run();
        publicVar.setValue("B", "BB");
    }
}


//测试类
public class Test {

    public static void main(String[] args) {
        try {
            //临界资源
            PublicVar publicVarRef = new PublicVar();

            //创建并启动线程
            ThreadA thread = new ThreadA(publicVarRef);
            thread.start();

            Thread.sleep(200);// 打印结果受此值大小影响

            //在主线程中调用
            publicVarRef.getValue();

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
/* Output ( 数据交叉 ): 
        method=getValue     threadName=main         username=B, password=AA
        method=setValue     threadName=Thread-0     username=B, password=BB
*///:~
```

​	由程序输出可知，虽然在写操作进行了同步，但在读操作上仍然有可能出现一些意想不到的情况，例如上面所示的 **脏读**。发生 脏读 的情况是在执行读操作时，相应的数据已被其他线程 **部分修改** 过，导致 **数据交叉** 的现象产生。

​	这其实就是一个线程安全问题，即多个线程同时访问一个资源时，会导致程序运行结果并不是想看到的结果。这里面，这个资源被称为：**临界资源**。也就是说，当多个线程同时访问临界资源（一个对象，对象中的属性，一个文件，一个数据库等）时，就可能会产生线程安全问题。

​	不过，**当多个线程执行一个方法时，该方法内部的局部变量并不是临界资源，因为这些局部变量是在每个线程的私有栈中，因此不具有共享性，不会导致线程安全问题。**



## 如何解决线程安全问题

​	实际上，所有的并发模式在解决线程安全问题时，采用的方案都是 **序列化访问临界资源** 。即在同一时刻，只能有一个线程访问临界资源，也称作 **同步互斥访问**。换句话说，就是在访问临界资源的代码前面加上一个锁，当访问完临界资源后释放锁，让其他线程继续访问。

​	**在 Java 中，提供了两种方式来实现同步互斥访问：synchronized 和 Lock。**



## synchronized 同步方法或者同步块

​	在了解 synchronized 关键字的使用方法之前，我们先来看一个概念：**互斥锁**，即 **能到达到互斥访问目的的锁。**举个简单的例子，如果对临界资源加上互斥锁，当一个线程在访问该临界资源时，其他线程便只能等待。

​	在 Java 中，可以使用 synchronized 关键字来标记一个方法或者代码块，当某个线程调用该对象的synchronized方法或者访问synchronized代码块时，这个线程便获得了该对象的锁，其他线程暂时无法访问这个方法，只有等待这个方法执行完毕或者代码块执行完毕，这个线程才会释放该对象的锁，其他线程才能执行这个方法或者代码块。

​	下面这段代码中两个线程分别调用insertData对象插入数据：

### **1) synchronized方法**

``` java
public class Test {

    public static void main(String[] args)  {
        final InsertData insertData = new InsertData();
        // 启动线程 1  
        new Thread() {
            public void run() {
                insertData.insert(Thread.currentThread());
            };
        }.start();

        // 启动线程 2
        new Thread() {
            public void run() {
                insertData.insert(Thread.currentThread());
            };
        }.start();
    }  
}

class InsertData {

    // 共享、可变资源
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();

    //对共享可变资源的访问
    public void insert(Thread thread){
        for(int i=0;i<5;i++){
            System.out.println(thread.getName()+"在插入数据"+i);
            arrayList.add(i);
        }
    }
}
/* Output: 
        Thread-0在插入数据0
        Thread-1在插入数据0
        Thread-0在插入数据1
        Thread-0在插入数据2
        Thread-1在插入数据1
        Thread-1在插入数据2
*///:~
```

​	根据运行结果就可以看出，这两个线程在同时执行insert()方法。而如果在insert()方法前面加上关键字synchronized 的话，运行结果为：

``` java
class InsertData {
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();

    public synchronized void insert(Thread thread){
        for(int i=0;i<5;i++){
            System.out.println(thread.getName()+"在插入数据"+i);
            arrayList.add(i);
        }
    }
}
/* Output: 
        Thread-0在插入数据0
        Thread-0在插入数据1
        Thread-0在插入数据2
        Thread-1在插入数据0
        Thread-1在插入数据1
        Thread-1在插入数据2
 *///:~
```

​	从以上输出结果可以看出，Thread-1 插入数据是等 Thread-0 插入完数据之后才进行的。说明 Thread-0 和 Thread-1 是顺序执行 insert() 方法的。这就是 synchronized 关键字对方法的作用。

​	不过需要注意以下三点：

​	1）**当一个线程正在访问一个对象的 synchronized 方法，那么其他线程不能访问该对象的其他 synchronized 方法。**这个原因很简单，因为一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他synchronized方法。

​	2）**当一个线程正在访问一个对象的 synchronized 方法，那么其他线程能访问该对象的非 synchronized 方法。**这个原因很简单，访问非 synchronized 方法不需要获得该对象的锁，假如一个方法没用 synchronized 关键字修饰，说明它不会使用到临界资源，那么其他线程是可以访问这个方法的，

​	　3）**如果一个线程 A 需要访问对象 object1 的 synchronized 方法 fun1，另外一个线程 B 需要访问对象 object2 的 synchronized 方法 fun1，即使 object1 和 object2 是同一类型），也不会产生线程安全问题，因为他们访问的是不同的对象，所以不存在互斥问题。**



### 2) synchronized 同步块

​	synchronized 代码块类似于以下这种形式：

``` java
	synchronized (lock){
   	 	//访问共享可变资源
   	 	...
	}
```

​	当在某个线程中执行这段代码块，该线程会获取对象lock的锁，从而使得其他线程无法同时访问该代码块。其中，lock 可以是 this，代表获取当前对象的锁，也可以是类中的一个属性，代表获取该属性的锁。特别地， **实例同步方法 与 synchronized(this)同步块是互斥的，因为它们锁的是同一个对象。但与 synchronized(非this)同步块 是异步的，因为它们锁的是不同对象。**

​	比如上面的insert()方法可以改成以下两种形式：

``` java
// this 监视器
class InsertData {
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();

    public void insert(Thread thread){
        synchronized (this) {
            for(int i=0;i<100;i++){
                System.out.println(thread.getName()+"在插入数据"+i);
                arrayList.add(i);
            }
        }
    }
}

// 对象监视器
class InsertData {
    private ArrayList<Integer> arrayList = new ArrayList<Integer>();
    private Object object = new Object();

    public void insert(Thread thread){
        synchronized (object) {
            for(int i=0;i<100;i++){
                System.out.println(thread.getName()+"在插入数据"+i);
                arrayList.add(i);
            }
        }
    }
}
```

​	从上面代码可以看出，**synchronized代码块 比 synchronized方法 的粒度更细一些，使用起来也灵活得多。**因为也许一个方法中只有一部分代码只需要同步，如果此时对整个方法用synchronized进行同步，会影响程序执行效率。而使用synchronized代码块就可以避免这个问题，synchronized代码块可以实现只对需要同步的地方进行同步。



### 3) class 对象锁

​	**特别地，每个类也会有一个锁，静态的 synchronized方法 就是以Class对象作为锁。** **另外，它可以用来控制对 static 数据成员（static 数据成员不专属于任何一个对象，是类成员） 的并发访问。**并且，如果一个线程执行一个对象的非static synchronized 方法，另外一个线程需要执行这个对象所属类的 static synchronized 方法，也不会发生互斥现象。因为访问 static synchronized 方法占用的是类锁，而访问非 static synchronized 方法占用的是对象锁，所以不存在互斥现象。例如:

``` java
public class Test {

    public static void main(String[] args)  {
        final InsertData insertData = new InsertData();
        new Thread(){
            @Override
            public void run() {
                insertData.insert();
            }
        }.start(); 
        new Thread(){
            @Override
            public void run() {
                insertData.insert1();
            }
        }.start();
    }  
}

class InsertData { 

    // 非 static synchronized 方法
    public synchronized void insert(){
        System.out.println("执行insert");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("执行insert完毕");
    }

    // static synchronized 方法
    public synchronized static void insert1() {
        System.out.println("执行insert1");
        System.out.println("执行insert1完毕");
    }
}
/* Output: 
        执行insert
        执行insert1
        执行insert1完毕
        执行insert完毕
*///:~
```

​	根据执行结果，我们可以看到第一个线程里面执行的是insert方法，不会导致第二个线程执行insert1方法发生阻塞现象。下面，我们看一下 synchronized 关键字到底做了什么事情，我们来反编译它的字节码看一下，下面这段代码反编译后的字节码为：

``` java
public class InsertData {
    private Object object = new Object();

    public void insert(Thread thread){
        synchronized (object) {}
    }

    public synchronized void insert1(Thread thread){}

    public void insert2(Thread thread){}
}
```

![192052201909119.jpg-81.2kB](C:\Users\12084\Desktop\JAVA多线程开发\反编译sychronized.jpg)

​	从反编译获得的字节码可以看出，**synchronized 代码块实际上多了 monitorenter 和 monitorexit 两条指令。** monitorenter指令执行时会让对象的锁计数加1，而monitorexit指令执行时会让对象的锁计数减1，其实这个与操作系统里面的PV操作很像，操作系统里面的PV操作就是用来控制多个进程对临界资源的访问。对于synchronized方法，执行中的线程识别该方法的 method_info 结构是否有 ACC_SYNCHRONIZED 标记设置，然后它自动获取对象的锁，调用方法，最后释放锁。如果有异常发生，线程自动释放锁。

​	有一点要注意：**对于 synchronized方法 或者 synchronized代码块，当出现异常时，JVM会自动释放当前线程占用的锁，因此不会由于异常导致出现死锁现象。**



## 可重入性

​	一般地，当某个线程请求一个由其他线程持有的锁时，发出请求的线程就会阻塞。然而，由于 Java 的内置锁是可重入的，因此如果某个线程试图获得一个已经由它自己持有的锁时，那么这个请求就会成功。**可重入锁最大的作用是避免死锁。**例如：

``` java
public class Test implements Runnable {

    // 可重入锁测试
    public synchronized void get() {
        System.out.println(Thread.currentThread().getName());
        set();
    }

    public synchronized void set() {
        System.out.println(Thread.currentThread().getName());
    }

    @Override
    public void run() {
        get();
    }

    public static void main(String[] args) {
        Test test = new Test();
        new Thread(test,"Thread-0").start();
        new Thread(test,"Thread-1").start();
        new Thread(test,"Thread-2").start();
    }
}
/* Output: 
        Thread-0
        Thread-0
        Thread-1
        Thread-1
        Thread-2
        Thread-2
 *///:~
```



## 注意事项

### **1). 内置锁与字符串常量**

​	**由于字符串常量池的原因，在大多数情况下，同步synchronized代码块 都不使用 String 作为锁对象**，而改用其他，比如 new Object() 实例化一个 Object 对象，因为它并不会被放入缓存中。看下面的例子：

``` java
//资源类
class Service {
    public void print(String stringParam) {
        try {
            synchronized (stringParam) {
                while (true) {
                    System.out.println(Thread.currentThread().getName());
                    Thread.sleep(1000);
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

//线程A
class ThreadA extends Thread {
    private Service service;

    public ThreadA(Service service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        service.print("AA");
    }
}

//线程B
class ThreadB extends Thread {
    private Service service;

    public ThreadB(Service service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        service.print("AA");
    }
}

//测试
public class Run {
    public static void main(String[] args) {

        //临界资源
        Service service = new Service();

        //创建并启动线程A
        ThreadA a = new ThreadA(service);
        a.setName("A");
        a.start();

        //创建并启动线程B
        ThreadB b = new ThreadB(service);
        b.setName("B");
        b.start();

    }
}
/* Output (死锁): 
        A
        A
        A
        A
        ...
 *///:~
```

​	出现上述结果就是因为 String 类型的参数都是 “AA”，两个线程持有相同的锁，所以 线程B 始终得不到执行，造成死锁。进一步地，所谓死锁是指：

​	**不同的线程都在等待根本不可能被释放的锁，从而导致所有的任务都无法继续完成。**

### **2). 锁的是对象而非引用**

在将任何数据类型作为同步锁时，需要注意的是，是否有多个线程将同时去竞争该锁对象： 

- 若它们将同时竞争同一把锁，则这些线程之间就是同步的； 
- 否则，这些线程之间就是异步的。

看下面的例子：

``` java
//资源类
class MyService {
    private String lock = "123";

    public void testMethod() {
        try {
            synchronized (lock) {
                System.out.println(Thread.currentThread().getName() + " begin "
                        + System.currentTimeMillis());
                lock = "456";
                Thread.sleep(2000);
                System.out.println(Thread.currentThread().getName() + "   end "
                        + System.currentTimeMillis());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

//线程B
class ThreadB extends Thread {

    private MyService service;

    public ThreadB(MyService service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        service.testMethod();
    }
}

//线程A
class ThreadA extends Thread {

    private MyService service;

    public ThreadA(MyService service) {
        super();
        this.service = service;
    }

    @Override
    public void run() {
        service.testMethod();
    }
}

//测试
public class Run1 {
    public static void main(String[] args) throws InterruptedException {

        //临界资源
        MyService service = new MyService();

        //线程A
        ThreadA a = new ThreadA(service);
        a.setName("A");

        //线程B
        ThreadB b = new ThreadB(service);
        b.setName("B");

        a.start();
        Thread.sleep(50);// 存在50毫秒
        b.start();
    }
}
/* Output(循环): 
       A begin 1484319778766
       B begin 1484319778815
       A   end 1484319780766
       B   end 1484319780815
*///:~
```

​	由上述结果可知，线程 A、B 是异步的。因为50毫秒过后， 线程B 取得的锁对象是 “456”，而 线程A 依然持有的锁对象是 “123”。所以，这两个线程是异步的。若将上述语句 “Thread.sleep(50);” 注释，则有：

``` java
//测试
public class Run1 {
    public static void main(String[] args) throws InterruptedException {

        //临界资源
        MyService service = new MyService();

        //线程A
        ThreadA a = new ThreadA(service);
        a.setName("A");

        //线程B
        ThreadB b = new ThreadB(service);
        b.setName("B");

        a.start();
        // Thread.sleep(50);// 存在50毫秒
        b.start();
    }
}
/* Output(循环): 
       B begin 1484319952017
       B   end 1484319954018
       A begin 1484319954018
       A   end 1484319956019
 *///:~
```

​	由上述结果可知，线程 A、B 是同步的。因为线程 A、B 竞争的是同一个锁“123”，虽然先获得运行的线程将 lock 指向了 对象“456”，但结果还是同步的。因为线程 A 和 B 共同争抢的锁对象是“123”，也就是说，**锁的是对象而非引用。**

## 总结

​	用一句话来说，**synchronized 内置锁 是一种 对象锁 (锁的是对象而非引用)， 作用粒度是对象 ，可以用来实现对 临界资源的同步互斥访问 ，是 可重入 的。**特别地，对于 **临界资源** 有：

- **若该资源是静态的，即被 static 关键字修饰，那么访问它的方法必须是同步且是静态的，synchronized 块必须是 class锁；**
- **若该资源是非静态的，即没有被 static 关键字修饰，那么访问它的方法必须是同步的，synchronized 块是实例对象锁；** 

**实质上，关键字synchronized 主要包含两个特征：**

- **互斥性：**保证在同一时刻，只有一个线程可以执行某一个方法或某一个代码块；
- **可见性：**保证线程工作内存中的变量与公共内存中的变量同步，使多线程读取共享变量时可以获得最新值的使用。