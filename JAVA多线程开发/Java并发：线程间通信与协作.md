# Java 并发：线程间通信与协作

​	线程与线程之间不是相互独立的个体，它们彼此之间需要相互通信和协作，最典型的例子就是生产者-消费者问题。本文首先介绍 wait/notify 机制，并对实现该机制的两种方式——synchronized+wait-notify模式和Lock+Condition模式进行详细剖析，以作为线程间通信与协作的基础。进一步地，以经典的生产者-消费者问题为背景，熟练对 wait/notify 机制的使用。最后，对 Thread 类中的 join() 方法进行源码分析，并以宿主线程与寄生线程的协作为例进行说明。

## 引子

​	线程与线程之间不是相互独立的个体，它们彼此之间需要相互通信和协作。比如说最经典的 **生产者-消费者模型：**当队列满时，生产者需要等待队列有空间才能继续往里面放入商品，而在等待的期间内，生产者必须释放对临界资源（即队列）的占用权。因为生产者如果不释放对临界资源的占用权，那么消费者就无法消费队列中的商品，就不会让队列有空间，那么生产者就会一直无限等待下去。因此，一般情况下，当队列满时，会让生产者交出对临界资源的占用权，并进入挂起状态。然后等待消费者消费了商品，然后消费者通知生产者队列有空间了。同样地，当队列空时，消费者也必须等待，等待生产者通知它队列中有商品了。这种互相通信的过程就是线程间的协作，也是本文要探讨的问题。

​	在下面的例子中，虽然两个线程实现了通信，但是凭借 线程B 不断地通过 **while语句轮询** 来检测某一个条件，这样会导致CPU的浪费。因此，需要一种机制来减少 CPU资源 的浪费，而且还能实现多个线程之间的通信，即 **wait/notify 机制** 。

``` java
//资源类
class MyList {

	//临界资源
	private volatile List<String> list = new ArrayList<String>();

	public void add() {
		list.add("abc");
	}

	public int size() {
		return list.size();
	}
}

// 线程A
class ThreadA extends Thread {

	private MyList list;

	public ThreadA(MyList list,String name) {
		super(name);
		this.list = list;
	}

	@Override
	public void run() {
		try {
			for (int i = 0; i < 3; i++) {
				list.add();
				System.out.println("添加了" + (i + 1) + "个元素");
				Thread.sleep(1000);
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

//线程B
class ThreadB extends Thread {

	private MyList list;

	public ThreadB(MyList list,String name) {
		super(name);
		this.list = list;
	}

	@Override
	public void run() {
		try {
			while (true) {          // while 语句轮询
				if (list.size() == 2) {
					System.out.println("==2了，线程b要退出了！");
					throw new InterruptedException();
				}
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

//测试
public class Test {
	public static void main(String[] args) {
		
		MyList service = new MyList();

		ThreadA a = new ThreadA(service,"A");
		ThreadB b = new ThreadB(service,"B");
		
		a.start();
		b.start();
	}
}
/* Output(输出结果不唯一): 
        添加了1个元素
        添加了2个元素
        ==2了，线程b要退出了！
        java.lang.InterruptedException
	        at test.ThreadB.run(Test.java:57)
        添加了3个元素
 *///:~

```



##  wait/notify 机制

​	在这之前，线程间通过共享数据来实现通信，即多个线程主动地读取一个共享数据，通过 **同步互斥访问机制** 保证线程的安全性。**等待/通知机制** 主要由 Object类 中的以下三个方法保证：

### 1、wait()、notify() 和 notifyAll()

​	上述三个方法均非Thread类中所声明的方法，而是Object类中声明的方法。原因是每个对象都拥有monitor（锁），所以让当前线程等待某个对象的锁，当然应该通过这个对象来操作，而不是用当前线程来操作，因为当前线程可能会等待多个线程的锁，如果通过线程来操作，就非常复杂了。

- **wait()**

  **让 当前线程 (Thread.concurrentThread() 方法所返回的线程) 释放对象锁并进入等待（阻塞）状态。**

- **notify()**

  **唤醒一个正在等待相应对象锁的线程，使其进入就绪队列，以便在当前线程释放锁后竞争锁，进而得到CPU的执行。**

- **notifyAll()**

  **唤醒所有正在等待相应对象锁的线程，使它们进入就绪队列，以便在当前线程释放锁后竞争锁，进而得到CPU的执行。**

总结来说：

- wait()、notify() 和 notifyAll()方法是 **本地方法**，并且为 **final 方法**，无法被重写；
- 调用某个对象的 wait() 方法能让 **当前线程阻塞**，并且当前线程必须拥有此对象的monitor（即锁）；
- 调用某个对象的 notify() 方法能够唤醒 **一个正在等待这个对象的monitor的线程**，如果有多个线程都在等待这个对象的monitor，则只能唤醒其中一个线程；
- 调用notifyAll()方法能够唤醒所有正在等待这个对象的monitor的线程。



### 2、方法调用与线程状态关系

​	**每个锁对象都有两个队列，一个是就绪队列，一个是阻塞队列。**就绪队列存储了已就绪（将要竞争锁）的线程，阻塞队列存储了被阻塞的线程。当一个阻塞线程被唤醒后，才会进入就绪队列，进而等待CPU的调度；反之，当一个线程被wait后，就会进入阻塞队列，等待被唤醒。

![](C:\Users\12084\Desktop\JAVA多线程开发\Thread方法与线程状态.jpg)

### 3、使用举例

``` java
public class Test {
	public static Object object = new Object();

	public static void main(String[] args) throws InterruptedException {
		Thread1 thread1 = new Thread1();
		Thread2 thread2 = new Thread2();

		thread1.start();

		Thread.sleep(2000);

		thread2.start();
	}

	static class Thread1 extends Thread {
		@Override
		public void run() {
			synchronized (object) {
				System.out.println("线程" + Thread.currentThread().getName()
						+ "获取到了锁...");
				try {
					System.out.println("线程" + Thread.currentThread().getName()
							+ "阻塞并释放锁...");
					object.wait();
				} catch (InterruptedException e) {
				}
				System.out.println("线程" + Thread.currentThread().getName()
						+ "执行完成...");
			}
		}
	}

	static class Thread2 extends Thread {
		@Override
		public void run() {
			synchronized (object) {
				System.out.println("线程" + Thread.currentThread().getName()
						+ "获取到了锁...");
				object.notify();
				System.out.println("线程" + Thread.currentThread().getName()
						+ "唤醒了正在wait的线程...");
			}
			System.out
					.println("线程" + Thread.currentThread().getName() + "执行完成...");
		}
	}
}
/* Output: 
        线程Thread-0获取到了锁...
        线程Thread-0阻塞并释放锁...
        线程Thread-1获取到了锁...
        线程Thread-1唤醒了正在wait的线程...
        线程Thread-1执行完成...
        线程Thread-0执行完成...
 *///:~

```

### **4、多个同类型线程的场景（wait 的条件发生变化）**

``` java
//资源类
class ValueObject {
	public static List<String> list = new ArrayList<String>();
}

//元素添加线程
class ThreadAdd extends Thread {

	private String lock;

	public ThreadAdd(String lock,String name) {
		super(name);
		this.lock = lock;
	}

	@Override
	public void run() {
		synchronized (lock) {
			ValueObject.list.add("anyString");
			lock.notifyAll();               // 唤醒所有 wait 线程
		}
	}
}

//元素删除线程
class ThreadSubtract extends Thread {

	private String lock;

	public ThreadSubtract(String lock,String name) {
		super(name);
		this.lock = lock;
	}

	@Override
	public void run() {
		try {
			synchronized (lock) {
				if (ValueObject.list.size() == 0) {
					System.out.println("wait begin ThreadName=" + Thread.currentThread().getName());
					lock.wait();
					System.out.println("wait   end ThreadName=" + Thread.currentThread().getName());
				}
				ValueObject.list.remove(0);
				System.out.println("list size=" + ValueObject.list.size());
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

//测试类
public class Run {
	public static void main(String[] args) throws InterruptedException {

		//锁对象
		String lock = new String("");

		ThreadSubtract subtract1Thread = new ThreadSubtract(lock,"subtract1Thread");
		subtract1Thread.start();

		ThreadSubtract subtract2Thread = new ThreadSubtract(lock,"subtract2Thread");
		subtract2Thread.start();

		Thread.sleep(1000);

		ThreadAdd addThread = new ThreadAdd(lock,"addThread");
		addThread.start();

	}
}
/* Output: 
        wait begin ThreadName=subtract1Thread
        wait begin ThreadName=subtract2Thread
        wait   end ThreadName=subtract2Thread
        list size=0
        wait   end ThreadName=subtract1Thread
        Exception in thread "subtract1Thread"
            java.lang.IndexOutOfBoundsException: Index: 0, Size: 0
	        at java.util.ArrayList.rangeCheck(Unknown Source)
        	at java.util.ArrayList.remove(Unknown Source)
        	at test.ThreadSubtract.run(Run.java:49)
 *///:~

```

​	当线程subtract1Thread 被唤醒后，将从 wait处继续执行。但由于 线程subtract2Thread 先获取到锁得到运行，导致线程subtract1Thread 的 wait的条件发生变化（不再满足），而线程subtract1Thread 却毫无所知，导致异常产生。

​	像这种有多个相同类型的线程场景，为防止wait的条件发生变化而导致的线程异常终止，我们在阻塞线程被唤醒的同时还必须对wait的条件进行额外的检查，如下所示：

``` java
//元素删除线程
class ThreadSubtract extends Thread {

	private String lock;

	public ThreadSubtract(String lock,String name) {
		super(name);
		this.lock = lock;
	}

	@Override
	public void run() {
		try {
			synchronized (lock) {
				while (ValueObject.list.size() == 0) {    //将 if 改成 while 
		System.out.println("wait begin ThreadName=" + Thread.currentThread().getName());
					lock.wait();
		System.out.println("wait   end ThreadName=" + Thread.currentThread().getName());
				}
				ValueObject.list.remove(0);
				System.out.println("list size=" + ValueObject.list.size());
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}/* Output: 
        wait begin ThreadName=subtract1Thread
        wait begin ThreadName=subtract2Thread
        wait   end ThreadName=subtract2Thread
        list size=0
        wait   end ThreadName=subtract1Thread
        wait begin ThreadName=subtract1Thread
 *///:~

```

**只需将 线程类ThreadSubtract 的 run()方法中的 if 条件 改为 while 条件 即可。**



## Condition

​	Condition是在java 1.5中出现的，它用来替代传统的Object的wait()/notify()实现线程间的协作，它的使用依赖于 Lock，Condition、Lock 和 Thread 三者之间的关系如下图所示。**相比使用Object的wait()/notify()，使用Condition的await()/signal()这种方式能够更加安全和高效地实现线程间协作。Condition是个接口，基本的方法就是await()和signal()方法。** **Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition() 。 必须要注意的是，** **Condition 的 await()/signal() 使用都必须在lock保护之内，也就是说，必须在lock.lock()和lock.unlock之间才可以使用。**事实上，Conditon的await()/signal() 与 Object的wait()/notify() 有着天然的对应关系：

- Conditon中的await()对应Object的wait()；
- Condition中的signal()对应Object的notify()；
- Condition中的signalAll()对应Object的notifyAll()。

![è¿éåå¾çæè¿°](C:\Users\12084\Desktop\JAVA多线程开发\Condition.png)

使用Condition往往比使用传统的通知等待机制(Object的wait()/notify())要更灵活、高效，例如，我们可以使用多个Condition实现通知部分线程：

``` java
// 线程 A
class ThreadA extends Thread {
	private MyService service;
	public ThreadA(MyService service) {
		super();
		this.service = service;
	}
	@Override
	public void run() {
		service.awaitA();
	}
}
// 线程 B
class ThreadB extends Thread {
    private MyService service;
	public ThreadB(MyService service) {
		super();
		this.service = service;
	}
	@Override
	public void run() {
		service.awaitB();
	}
}

class MyService {
	private Lock lock = new ReentrantLock();
	// 使用多个Condition实现通知部分线程
	public Condition conditionA = lock.newCondition();
	public Condition conditionB = lock.newCondition();

	public void awaitA() {
	    lock.lock();
	    try {
		    System.out.println("begin awaitA时间为" + System.currentTimeMillis()
			    	+ " ThreadName=" + Thread.currentThread().getName());
		    conditionA.await();
		    System.out.println("  end awaitA时间为" + System.currentTimeMillis()
			    	+ " ThreadName=" + Thread.currentThread().getName());
	    } catch (InterruptedException e) {
		    e.printStackTrace();
	    } finally {
		    lock.unlock();
	    }
    }

	public void awaitB() {
	    lock.lock();
		try {
			System.out.println("begin awaitB时间为" + System.currentTimeMillis()
					+ " ThreadName=" + Thread.currentThread().getName());
			conditionB.await();
			System.out.println("  end awaitB时间为" + System.currentTimeMillis()
					+ " ThreadName=" + Thread.currentThread().getName());
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void signalAll_A() {
		try {
			lock.lock();
			System.out.println("  signalAll_A时间为" + System.currentTimeMillis()
					+ " ThreadName=" + Thread.currentThread().getName());
			conditionA.signalAll();
		} finally {
			lock.unlock();
		}
	}

	public void signalAll_B() {
		try {
			lock.lock();
			System.out.println("  signalAll_B时间为" + System.currentTimeMillis()
					+ " ThreadName=" + Thread.currentThread().getName());
			conditionB.signalAll();
		} finally {
			lock.unlock();
		}
	}
}

// 测试
public class Run {
	public static void main(String[] args) throws InterruptedException {
	    MyService service = new MyService();

	    ThreadA a = new ThreadA(service);
	    a.setName("A");
	    a.start();

    	ThreadB b = new ThreadB(service);
	    b.setName("B");
    	b.start();

	    Thread.sleep(3000);
	    service.signalAll_A();
    }
}
```

输出结果如下图所示，我们可以看到只有线程A被唤醒，线程B仍然阻塞。

![å¤ä¸ªConditionéç¥é¨åçº¿ç¨.png-13.1kB](C:\Users\12084\Desktop\JAVA多线程开发\Condition实例.png)

​	实际上，**Condition 实现了一种分组机制，将所有对临界资源进行访问的线程进行分组，以便实现线程间更精细化的协作，例如通知部分线程。**我们可以从上面例子的输出结果看出，只有conditionA范围内的线程A被唤醒，而conditionB范围内的线程B仍然阻塞。



## 生产者-消费者模型

​	**等待/通知机制** 最经典的应用就是 **生产者-消费者模型**。下面以多生产者-多消费者问题为背景，分别运用两种模式 —— synchronized+wait-notify模式和Lock+Condition模式实现 wait-notify 机制。



### **Case 1： 传统实现方式**

``` java
//资源类
class MyStack {
    // 共享队列
	private List list = new ArrayList();

    // 生产
	@SuppressWarnings("unchecked")
	public synchronized void push() {
		try {
			while (list.size() == 1) {    // 多个生产者
				System.out.println("队列已满，线程 "
						+ Thread.currentThread().getName() + " 呈wait状态...");
				this.wait();
			}
			list.add("anyString=" + Math.random());
			System.out.println("线程 " + Thread.currentThread().getName()
					+ " 生产了，队列已满...");
			this.notifyAll();                   // 防止生产者仅通知生产者
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
	
    // 消费
	public synchronized String pop() {
		String returnValue = "";
		try {
			while (list.size() == 0) {              // 多个消费者
				System.out.println("队列已空，线程 "
						+ Thread.currentThread().getName() + " 呈wait状态...");
				this.wait();
			}
			returnValue = "" + list.get(0);
			list.remove(0);
			System.out.println("线程 " + Thread.currentThread().getName()
					+ " 消费了，队列已空...");
			this.notifyAll();                   // 防止消费者仅通知消费者
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return returnValue;
	}
}

//生产者
class P_Thread extends Thread {

	private MyStack myStack;

	public P_Thread(MyStack myStack,String name) {
		super(name);
		this.myStack = myStack;
	}

	public void pushService() {
		myStack.push();
	}
	
	@Override
	public void run() {
		while (true) {     
			myStack.push();
		}
	}
}

//消费者
class C_Thread extends Thread {

	private MyStack myStack;

	public C_Thread(MyStack myStack,String name) {
		super(name);
		this.myStack = myStack;
	}

	@Override
	public void run() {
		while (true) {
			myStack.pop();
		}
	}
}

//测试类
public class Run {
	public static void main(String[] args) throws InterruptedException {
		MyStack myStack = new MyStack();

		P_Thread pThread1 = new P_Thread(myStack, "P1");
		P_Thread pThread2 = new P_Thread(myStack, "P2");
		P_Thread pThread3 = new P_Thread(myStack, "P3");
		P_Thread pThread4 = new P_Thread(myStack, "P4");
		P_Thread pThread5 = new P_Thread(myStack, "P5");
		P_Thread pThread6 = new P_Thread(myStack, "P6");
		pThread1.start();
		pThread2.start();
		pThread3.start();
		pThread4.start();
		pThread5.start();
		pThread6.start();

		C_Thread cThread1 = new C_Thread(myStack, "C1");
		C_Thread cThread2 = new C_Thread(myStack, "C2");
		C_Thread cThread3 = new C_Thread(myStack, "C3");
		C_Thread cThread4 = new C_Thread(myStack, "C4");
		C_Thread cThread5 = new C_Thread(myStack, "C5");
		C_Thread cThread6 = new C_Thread(myStack, "C6");
		C_Thread cThread7 = new C_Thread(myStack, "C7");
		C_Thread cThread8 = new C_Thread(myStack, "C8");
		cThread1.start();
		cThread2.start();
		cThread3.start();
		cThread4.start();
		cThread5.start();
		cThread6.start();
		cThread7.start();
		cThread8.start();
	}
}/* Output: 
        线程 P1 生产了，队列已满...
        队列已满，线程 P1 呈wait状态...
        线程 C5 消费了，队列已空...
        队列已空，线程 C5 呈wait状态...
        队列已空，线程 C8 呈wait状态...
        队列已空，线程 C2 呈wait状态...
        队列已空，线程 C7 呈wait状态...
        队列已空，线程 C4 呈wait状态...
        队列已空，线程 C6 呈wait状态...
        队列已空，线程 C3 呈wait状态...
        队列已空，线程 C1 呈wait状态...
        线程 P6 生产了，队列已满...
        队列已满，线程 P6 呈wait状态...
        队列已满，线程 P5 呈wait状态...
        队列已满，线程 P4 呈wait状态...
        ...
 *///:~
```

对于生产者-消费者问题，有两个要点需要注意：

- **在多个同类型线程（多个生产者线程或者消费者线程）的场景中，为防止wait的条件发生变化而导致线程异常终止，我们在阻塞线程被唤醒的同时还必须对wait的条件进行额外的检查，即 使用 while 循环代替 if 条件；**
- **在多个同类型线程（多个生产者线程或者消费者线程）的场景中，为防止生产者(消费者)唤醒生产者(消费者)，保证生产者和消费者互相唤醒，需要 使用 notify 替代 notifyAll.**



### **Case 2： 使用 Condition 实现方式**

``` java
// 线程A
class MyThreadA extends Thread {

	private MyService myService;

	public MyThreadA(MyService myService, String name) {
		super(name);
		this.myService = myService;
	}

	@Override
	public void run() {
		while (true)
			myService.set();
	}
}

// 线程B
class MyThreadB extends Thread {

	private MyService myService;

	public MyThreadB(MyService myService, String name) {
		super(name);
		this.myService = myService;
	}

	@Override
	public void run() {
		while (true)
			myService.get();
	}
}

// 资源类
class MyService {

	private ReentrantLock lock = new ReentrantLock();
	private Condition conditionA = lock.newCondition();   // 生产线程
	private Condition conditionB = lock.newCondition();  // 消费线程
	private boolean hasValue = false;

	public void set() {
		try {
			lock.lock();
			while (hasValue == true) {
				System.out.println("[生产线程] " + " 线程"
						+ Thread.currentThread().getName() + " await...");
				conditionA.await();
			}
			System.out.println("[生产中] " + " 线程" + Thread.currentThread().getName() + " 生产★");
			Thread.sleep(1000);
			hasValue = true;
			System.out.println("线程" + Thread.currentThread().getName() + " 生产完毕...");
			System.out.println("[唤醒所有消费线程] " + " 线程"
					+ Thread.currentThread().getName() + "...");
			conditionB.signalAll();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void get() {
		try {
			lock.lock();
			while (hasValue == false) {
				System.out.println("[消费线程] " + " 线程"
						+ Thread.currentThread().getName() + " await...");
				conditionB.await();
			}
			System.out.println("[消费中] " + " 线程"
					+ Thread.currentThread().getName() + " 消费☆");
			Thread.sleep(1000);
			System.out.println("线程" + Thread.currentThread().getName() + " 消费完毕...");
			hasValue = false;
			System.out.println("[唤醒所有生产线程] " + " 线程"
					+ Thread.currentThread().getName() + "...");
			conditionA.signalAll();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
}

public class Run {
	public static void main(String[] args) throws InterruptedException {
		MyService service = new MyService();

		MyThreadA[] threadA = new MyThreadA[10];
		MyThreadB[] threadB = new MyThreadB[10];

		for (int i = 0; i < 10; i++) {
			threadA[i] = new MyThreadA(service, "ThreadA-" + i);
			threadB[i] = new MyThreadB(service, "ThreadB-" + i);
			threadA[i].start();
			threadB[i].start();
		}
	}
}
/* Output: 
        [生产中]  线程ThreadA-0 生产★
        线程ThreadA-0 生产完毕...
        [唤醒所有消费线程]  线程ThreadA-0...
        [生产线程]  线程ThreadA-0 await...
        [消费中]  线程ThreadB-0 消费☆
        线程ThreadB-0 消费完毕...
        [唤醒所有生产线程]  线程ThreadB-0...
        [消费线程]  线程ThreadB-0 await...
        [生产中]  线程ThreadA-1 生产★
        线程ThreadA-1 生产完毕...
        [唤醒所有消费线程]  线程ThreadA-1...
        [生产线程]  线程ThreadA-1 await...
        [消费中]  线程ThreadB-1 消费☆
        线程ThreadB-1 消费完毕...
        [唤醒所有生产线程]  线程ThreadB-1...
        [消费线程]  线程ThreadB-1 await...
        [生产中]  线程ThreadA-2 生产★
        线程ThreadA-2 生产完毕...
        [唤醒所有消费线程]  线程ThreadA-2...
        ...
 *///:~
```



## 线程间的通信：管道

​	PipedInputStream类 与 PipedOutputStream类 用于在应用程序中创建管道通信。一个PipedInputStream实例对象必须和一个PipedOutputStream实例对象进行连接而产生一个通信管道。PipedOutputStream可以向管道中写入数据，PipedIntputStream可以读取PipedOutputStream向管道中写入的数据，这两个类主要用来完成线程之间的通信。一个线程的PipedInputStream对象能够从另外一个线程的PipedOutputStream对象中读取数据，如下图所示：

![çº¿ç¨éä¿¡ç¤ºæå¾ä¹ç®¡é.jpg-30.5kB](C:\Users\12084\Desktop\JAVA多线程开发\管道.jpg)

​	PipedInputStream和PipedOutputStream的实现原理类似于"生产者-消费者"原理，PipedOutputStream是生产者，PipedInputStream是消费者。在PipedInputStream中，有一个buffer字节数组，默认大小为1024，作为缓冲区，存放"生产者"生产出来的东西。此外，还有两个变量in和out —— in用来记录"生产者"生产了多少，out是用来记录"消费者"消费了多少，in为-1表示消费完了，in==out表示生产满了。当消费者没东西可消费的时候，也就是当in为-1的时候，消费者会一直等待，直到有东西可消费。

​	在 Java 的 JDK 中，提供了四个类用于线程间通信：

- 字节流：PipedInputStream 和 PipedOutputStream;
- 字符流：PipedReader 和 PipedWriter;

``` java
//读线程
class ThreadRead extends Thread {

	private ReadData read;
	private PipedInputStream input;

	public ThreadRead(ReadData read, PipedInputStream input) {
		super();
		this.read = read;
		this.input = input;
	}

	public void readMethod(PipedInputStream input) {
		try {
			System.out.println("read  :");
			byte[] byteArray = new byte[20];
			int readLength = input.read(byteArray);
			while (readLength != -1) {
				String newData = new String(byteArray, 0, readLength);
				System.out.print(newData);
				readLength = input.read(byteArray);
			}
			System.out.println();
			input.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	@Override
	public void run() {
		this.readMethod(input);
	}
}

//写线程
class ThreadWrite extends Thread {

	private WriteData write;
	private PipedOutputStream out;

	public ThreadWrite(WriteData write, PipedOutputStream out) {
		super();
		this.write = write;
		this.out = out;
	}

	public void writeMethod(PipedOutputStream out) {
		try {
			System.out.println("write :");
			for (int i = 0; i < 30; i++) {
				String outData = "" + (i + 1);
				out.write(outData.getBytes());
				System.out.print(outData);
			}
			System.out.println();
			out.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	@Override
	public void run() {
		this.writeMethod(out);
	}
}

//测试
public class Run {

	public static void main(String[] args) {
		try {
			WriteData writeData = new WriteData();
			ReadData readData = new ReadData();

			PipedInputStream inputStream = new PipedInputStream();
			PipedOutputStream outputStream = new PipedOutputStream();

			// inputStream.connect(outputStream);   // 效果相同
			outputStream.connect(inputStream);

			ThreadRead threadRead = new ThreadRead(readData, inputStream);
			threadRead.start();

			Thread.sleep(2000);

			ThreadWrite threadWrite = new ThreadWrite(writeData, outputStream);
			threadWrite.start();

		} catch (IOException e) {
			e.printStackTrace();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}/* Output: 
        read  :
        write :
        123456789101112131415161718192021222324252627282930
        123456789101112131415161718192021222324252627282930
 *///:~
```



## 方法 join() 的使用

### 1). join() 的定义

​	**假如在main线程中调用thread.join方法，则main线程会等待thread线程执行完毕或者等待一定的时间。**详细地，如果调用的是无参join方法，则等待thread执行完毕；如果调用的是指定了时间参数的join方法，则等待一定的时间。join()方法有三个重载版本：

``` java
public final synchronized void join(long millis) throws InterruptedException {...}
public final synchronized void join(long millis, int nanos) throws InterruptedException {...}
public final void join() throws InterruptedException {...}
```

以 join(long millis) 方法为例，其内部调用了Object的wait()方法，如下图：

![](C:\Users\12084\Desktop\JAVA多线程开发\join.png)

​	根据以上源代码可以看出，**join()方法是通过wait()方法 (Object 提供的方法) 实现的。** **当 millis == 0 时，会进入 while(isAlive()) 循环，并且只要子线程是活的，宿主线程就不停的等待。** **wait(0) 的作用是让当前线程(宿主线程)等待，而这里的当前线程是指 Thread.currentThread() 所返回的线程。所以，虽然是子线程对象(锁)调用wait()方法，但是阻塞的是宿主线程。**



### 2). join() 使用实例及原理

``` java
//示例代码
public class Test {
     
    public static void main(String[] args) throws IOException  {
        System.out.println("进入线程"+Thread.currentThread().getName());
        Test test = new Test();
        MyThread thread1 = test.new MyThread();
        thread1.start();
        try {
            System.out.println("线程"+Thread.currentThread().getName()+"等待");
            thread1.join();
            System.out.println("线程"+Thread.currentThread().getName()+"继续执行");
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    } 
     
    class MyThread extends Thread{
        @Override
        public void run() {
            System.out.println("进入线程"+Thread.currentThread().getName());
            try {
                Thread.currentThread().sleep(5000);
            } catch (InterruptedException e) {
                // TODO: handle exception
            }
            System.out.println("线程"+Thread.currentThread().getName()+"执行完毕");
        }
    }
}
 /* Output:
        进入线程main
        线程main等待
        进入线程Thread-0
        线程Thread-0执行完毕
        线程main继续执行
 *///~
```

​	看上面的例子，**当 main线程 运行到 thread1.join() 时，main线程会获得线程对象thread1的锁(wait 意味着拿到该对象的锁)。** **只要 thread1线程 存活， 就会调用该对象锁的wait()方法阻塞 main线程。那么，main线程被什么时候唤醒呢？**

​	事实上，**有wait就必然有notify。**在整个jdk里面，我们都不会找到对thread1线程的notify操作。这就要看jvm代码了：

``` java
//一个c++函数：
void JavaThread::exit(bool destroy_vm, ExitType exit_type) ；

//这个函数的作用就是在一个线程执行完毕之后，jvm会做的收尾工作。里面有一行代码：ensure_join(this);

该函数源码如下：

static void ensure_join(JavaThread* thread) {
    Handle threadObj(thread, thread->threadObj());

    ObjectLocker lock(threadObj, thread);

    thread->clear_pending_exception();

    java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);

    java_lang_Thread::set_thread(threadObj(), NULL);

    //thread就是当前线程，就是刚才说的thread1线程。
    lock.notify_all(thread);

    thread->clear_pending_exception();
}

```

至此，thread1线程对象锁调用了notifyall，那么main线程也就能继续跑下去了。

由于 join方法 会调用 wait方法 让宿主线程进入阻塞状态，并且会释放线程占有的锁，并交出CPU执行权限。结合 join 方法的声明，可以总结出以下三条：

- **join方法同样会会让线程交出CPU执行权限；**
- **join方法同样会让线程释放对一个对象持有的锁；**
- **如果调用了join方法，必须捕获InterruptedException异常或者将该异常向上层抛出。**



