# Java并发——Executor框架详解（Executor框架结构与框架成员）

## **什么是Executor框架？**

​	我们知道线程池就是线程的集合，线程池集中管理线程，以实现线程的重用，降低资源消耗，提高响应速度等。线程用于执行异步任务，单个的线程既是工作单元也是执行机制，从JDK1.5开始，为了把工作单元与执行机制分离开，Executor框架诞生了，他是一个用于统一创建与运行的接口。Executor框架实现的就是线程池的功能。

## **Executor框架结构图解**

### 1、Executor框架包括3大部分：

- 任务。也就是工作单元，包括被执行任务需要实现的接口：Runnable接口或者Callable接口；
- 任务的执行。也就是把任务分派给多个线程的执行机制，包括Executor接口及继承自Executor接口的ExecutorService接口。
- 异步计算的结果。包括Future接口及实现了Future接口的FutureTask类。

Executor框架的成员及其关系可以用一下的关系图表示：

![img](C:\Users\12084\Desktop\JAVA多线程开发\Executor.png)

### 2、Executor框架的使用示意图：

![img](C:\Users\12084\Desktop\JAVA多线程开发\Executor使用.png)

使用步骤：

（1）创建Runnable并重写run（）方法或者Callable对象并重写call（）方法：

``` java
class callableTest implements Callable<String >{
            @Override
            public String call() {
                try{
                    String a = "return String";
                    return a;
                }
                catch(Exception e){
                    e.printStackTrace();
                    return "exception";
                }
            }
        }
```

（2）创建Executor接口的实现类ThreadPoolExecutor类或者ScheduledThreadPoolExecutor类的对象，然后调用其execute（）方法或者submit（）方法把工作任务添加到线程中，如果有返回值则返回Future对象。其中Callable对象有返回值，因此使用submit（）方法；而Runnable可以使用execute（）方法，此外还可以使用submit（）方法，只要使用callable（Runnable task）或者callable(Runnable task,  Object result)方法把Runnable对象包装起来就可以，使用callable（Runnable task）方法返回的null，使用callable(Runnable task,  Object result)方法返回result。

``` java
ThreadPoolExecutor tpe = new ThreadPoolExecutor(5, 10,
                100, MILLISECONDS, new ArrayBlockingQueue<Runnable>(5));
Future<String> future = tpe.submit(new callableTest());
```

（3）调用Future对象的get（）方法后的返回值，或者调用Future对象的cancel（）方法取消当前线程的执行。最后关闭线程池

``` java
		try{
            System.out.println(future.get());
        }
        catch(Exception e){
            e.printStackTrace();
        }
        finally{
            tpe.shutdown();
        }
```

## **Executor框架成员**：

​	**ThreadPoolExecutor实现类、ScheduledThreadPoolExecutor实现类、Future接口、Runnable和Callable接口、Executors工厂类**

![img](C:\Users\12084\Desktop\JAVA多线程开发\Executor框架成员.png)

​	