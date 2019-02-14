# java线程池和中断总结

[TOC]

本系列文是对自己学习多线程和平时使用过程中的知识梳理，不适合基础比较差的阅读，适合看过java编程实战做整体回顾的，想到了会不断补充。

## 一、 线程池的使用

线程池其实在实际工作中有用到的话理解其实是非常简单的，合理的利用线程池能极大的提高效率。主要说明下程池的使用和参数的意义（暂时不考虑定时线程池）：

    1. corePoolSize 线程池的最小大小
    2. maximumPoolSize 线程池的最大大小
    3. keepAliveTime 大于线程池最小大小的空余线程的包活时间
    4. workQueue 工作队列用于存放任务
    5. threadFactory 创建线程的线程工厂
    6. handler  用于处理任务被拒绝的情况

```java
   public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

**线程池的任务投递过程**：

1. 向线程池投递一个任务时，首先看工作者线程有没有小于corePoolSize，如果是，利用threadFactory创建一个线程将任务投递
2. 第一步如果大于corePoolSize，则将任务投递到workQueue，这里需要考虑workQueue是无界还是有界的情况，如果是无界肯定投递成功返回。如果是有界，投递成功则返回，否则看线程数有没有小于maximumPoolSize，如果是则再开一个线程返回，不是的话则调用handler的拒绝逻辑。

**需要注意的点**：

1. 如果放的workQueue是无界队列，那么maximumPoolSize这个参数其实就无效了，永远不会创建超过corePoolSize的线程数量，所以任务永远不会因为容量问题被拒绝，如果生产者速度一直大于消费者，很可能造成内存溢出
2. 第1条说workQueue是无界队列那么任务永远不会因为容量问题被拒绝，但是handler还是有用的，当你关闭了线程池池，继续提交任务会用到这个来拒绝
3. 用户代码在提交任务时底层使用的阻塞队列的offer方法，所有一般是不会阻塞的，要么成功，要么被阻绝。

**关于线程池参数的设置**：   

corePoolSize，maximumPoolSize，workQueue核心参数：    
根据你的业务场景，如果是cpu密集型，可以设置线程池固定为ncpu+1，队列采用一个有界的，防止内存溢出。如果还是出现任务被拒绝是不是应该考虑加机器了。
如果是io密集型，需要根据io和cpu的比例来做相应的估算，这种也不是十分精确的，毕竟网络情况也会发生变化。这里推荐书中的公式： ncpu*ucpu*(1+w/c)  ncpu:cpu个数  ucpu:每个cpu利用率  w/c: io/cpu    
另外不同类型的任务最好采用不同的线程池有利于线程池的设置，混杂的任务很难设置。   

threadFactory:我主要用于设置线程的名字，这样在日志区分度更高

handler：拒绝执行器，这个得根据业务场景类    

**关于Executors的工具方法**：   

alibaba编码规约中建议是手动创建线程池，这样根据自己的业务做更好的定制，而且自己也能更加的清理里面的逻辑。如果一时图快，可能在系统的负载不断升高时出现问题，反而更加不好排查。


**关于线程池的优雅停机**：   

在提高性能的同时不要忘记数据的安全性，因为线程池的特点，任务会被缓存在队列中，如果是不重要的应用可以直接将线程设置成守护线程，这样在应用停机的时候直接停机。   
对于重要的应用，如果应用重启这些数据都是要考虑到的。这里就需要十分清楚中断机制，因为这里涉及任务取消的逻辑，这些逻辑是要对自己的任务代码自己进行相应的处理。线程池shutdown方法执行之后后续的任务都会被拒绝，已经有的任务会继续执行完，这个比较好理解。shutdownNow方法返回队列中的所有任务，然后发中断给正在执行的任务，这里返回的任务你可以进行持久化，主要就是正在执行的任务的处理，对于短任务你可以不响应中断，耗时任务必须得考虑进程退出时间过长被强杀。   


**实际应用场景举例**：   

1. 应用日志的记录：我们对于一些业务日志可能写到mysql中，如果每个操作插入一条日志必然会很耗时，这是我们可以单独开一个日志线程，将日志投递到日志线程对象的queue中，然后他定时扫，批量如库，既提高吞吐量，有降低延迟
2. 第三方接口对接：在对接第三方时候，很多时候是http，这种操作相当耗时，可以利用线程池来进行异步化，如果需要得到返回接口可以利用Feature和CompletableFuture（后续讲解）
3. 耗时的线程不安全操作：这种场景比较少，但公司确实遇到了，具体就是后台提交一个任务，这种任务可能会执行几十分钟，任务不能同时执行。这里思路就是采用单线程池，然后利用提交时返回的Future来实现任务的取消功能。


**关于异常**：

只有execute提交的任务才会将异常交给未捕获异常处理器，默认的未捕获异常处理器会打印异常。但是如果是submit会将异常封装到饭返回的Future中在get的时候才会抛出，可以通过改写线程池的afterExecute方法。   
源码中的例子：    

```java
protected void afterExecute(Runnable r, Throwable t) {
     super.afterExecute(r, t);
     if (t == null && r instanceof Future<?>) {
       try {
         Object result = ((Future<?>) r).get();
       } catch (CancellationException ce) {
           t = ce;
       } catch (ExecutionException ee) {
           t = ee.getCause();
       } catch (InterruptedException ie) {
           Thread.currentThread().interrupt(); // ignore/reset
       }
     }
     if (t != null)
       System.out.println(t);
   }
```

## 二、 java中断机制
    
中断机制对于javq多线程编程是一个十分基础的东西，很多时候都是在使用类库所以没有注意到，对于自己更好的使用类库和自己封装类库，中断是十分重要的。对于中断处理的好坏直接影响了编写的api合理性和使用类库时正确性。

在多线程中，很多时候会遇到需要停止一个线程中的任务这样的需求。实现这样的需求很容易想到在对象中放置一个标志位，然后线程在执行的过程中去不停的检测这个标志位，如果标志位被设置成false就退出。另外标志位需要采用volatile来修饰，可以保证内存的可见性。例子如下：

```java
public class Main {
    public static void main(String[] args) throws InterruptedException {
        //创建一个任务
        Task task = new Task();
        new Thread(task).start();
        Thread.sleep(3_000);
        task.stop = true;
    }
}

@Slf4j
class Task implements Runnable {
    /**
     * 是否停止的标志位
     */
    public volatile boolean stop = false;
    /**
     * 执行次数计数器
     */
    AtomicInteger adder = new AtomicInteger();

    @Override
    public void run() {
        while (!stop) {
            log.info("运行次数：{}", adder.incrementAndGet());
            try {
                Thread.sleep(1_000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.warn("退出运行！");
    }
}

```
上面的代码是很多人一开始就能想到的方案，但是，仔细想想就会发现问题，上面的代码中的sleep函数是一个类库封装好的，所以如果设置了停止标志位，那么每次检测运行都得等到while循环才行。这就是引入中断的意义，jdk中很多函数都能响应中断的操作。

先说一下java中断的含义：java中断是一种线程间协作机制，用来告诉一个线程可以停止了，但是具体那个线程是否响应以及采取什么样的动作，java语言没有做任何强制规定。这里就需要和操作系统的中的中断明确区别，这两种虽然中文名一样，但是实际的意义却差以千里。ava的中断是协作的，相当于只是告诉一个线程，但是那个线程可以选择不响应或者需要中断。

第二个版本的代码如下：

```java
public class MainInterrupt {
    public static void main(String[] args) throws InterruptedException {
        //创建一个任务
        Thread thread = new Thread(new TaskInterrupt());
        thread.start();
        Thread.sleep(3_000);
        thread.interrupt();
    }
}

@Slf4j
class TaskInterrupt implements Runnable {
    /**
     * 执行次数计数器
     */
    AtomicInteger adder = new AtomicInteger();

    @Override
    public void run() {
        while (!Thread.interrupted()) {
            log.info("运行次数：{}", adder.incrementAndGet());
            try {
                Thread.sleep(1_000);
            } catch (InterruptedException e) {
                log.warn("随眠过程打断退出！");
                break;
            }
        }
        log.warn("退出运行！");
    }
}
```

中断的api：      
1. public void interrupt()  给一个线程发起中断
2. public static boolean interrupted()  检测线程是否处于中断位，并清除标志位，**这也是唯一清除标志位的  方法**！
3. public boolean isInterrupted()  检测线程是否中断

### 中断的处理 

  封装的自己类库代码的时候一定要考虑清楚对于中断的处理，例如BlockingQueue的<code>E poll(long timeout, TimeUnit unit)throws InterruptedException</code>这个api，他的实现都是选择了将异常抛出，底层实现一般是Lock的<code>lockInterruptibly()</code>方法阻塞抛出。你站在用户代码的角度去想，如果你实现了这个方法是阻塞的，然后又把异常吃了，怎么去实现被取消后的逻辑，用户代码又怎么去针对取消做相应的动作。所以封装类库的时候最好还是重新抛出。   
  
  另外还有一种就是重置中断位，有些操作不能直接抛出，像Runnable接口，还有抛出异常前执行一些动作的情况。在处理完之后重置下中断位，这样就能让用户代码感知并做相应的处理。

## 三、 线程间通信机制总结

这里只是简单的罗列，当时面试的时候被问到线程间通信，当时没反应过来，其实都是知道的，这一块知道这么个东西比较简单，很多需要看源码的实现，看区别。以后会进行相关的源码分析

1. synchronize关键字
2. 对象的wait，notify，notifyall
3. Thread的join
4. Semaphore，ReentrantLock，CyclicBarrier，CountDownLatch，(这些都是基于AQS的工具类)BlockingQueue
5. FutureTask相关的
