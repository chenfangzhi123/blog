#java并发编程降低延迟

在平时开发的过程中，其实有很多可以采用多线程优化的地方，像ExecutorService、CompletionService、CompletableFuture和并行流等类，只是没有去注意，这里总结下日常工作中常用的一些方法。

[TOC]

## 一、ExecutorService和CompletionService

**基本的execute和submit方法**   

这个其实没有太多好说的，因为这个是最基本的，基本使用线程池的都会使用到这个方法，主要用于异步执行任务，submit和execute的区别就在于，submit有一个方法的回执，可以利用这个Future对这个任务的生命周期进行干预。

**invokeAll和invokeAny方法**
 
很多人没有注意到这两个方法，这两个方法其实也是非常有用的，例如你有很多可以并行执行的操作投递到线程池，执行完之后就挨个调用Future的get获取结果最后生成结果，这两个步骤其实就是invokeAll已经封装好的。他的内部实现也很简单和你手动取每个值是一样的，这个方法只会到所有任务执行完毕或者设定的时间超时了才会返回。实现非常简单：

```java
 for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) {
                    try {
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
}

```

invokeAny方法稍微比invokeAll复杂些，内部是基于ExecutorCompletionService实现的。如果有一个任务返回了就直接返回结果，如果第一个完成的任务抛出了异常那么这个方法会抛出对应的异常。

**CompletionService**

这个类名中文翻译就是完成服务，这个类组合了ExecutorService，实现逻辑非常简单，内部存放了一个阻塞队列，当投递的任务完成时会将对应的Future放入这个阻塞队列，这样就可以做到投递的任务在完成的顺序依次放入阻塞队列。这就是上面invokeAny实现利用主要逻辑。利用阻塞队列的poll和take方法，在第一个返回时就取消剩余的任务。    
虽然invokeAny已经封装了CompletionService的逻辑但是有些场景这个类还是很有用的。比如现在我想要得到一个最先完成的但是没有抛出异常的，这种情况下我们就需要写一个类似于invokeAny的例子。jdk注释中给出了例子：    
```java
void solve(Executor e,
            Collection<Callable<Result>> solvers)
     throws InterruptedException {
     CompletionService<Result> ecs
         = new ExecutorCompletionService<Result>(e);
     int n = solvers.size();
     List<Future<Result>> futures
         = new ArrayList<Future<Result>>(n);
     Result result = null;
     try {
         for (Callable<Result> s : solvers)
             futures.add(ecs.submit(s));
         for (int i = 0; i < n; ++i) {
             try {
                 Result r = ecs.take().get();
                 if (r != null) {
                     result = r;
                     break;
                 }
             } catch (ExecutionException ignore) {}
         }
     }
     finally {
         for (Future<Result> f : futures)
             f.cancel(true);
     }

     if (result != null)
         use(result);
 }
```

这个类也很容易想到一个场景，我有很多任务是可以并发执行了，这时可以使用invokeAll，但是让必须等到所有的任务执行完毕才能返回，这时如果有一个任务被io阻塞了很慢将会导致整个方法阻塞。如果是利用CompletionService的话，因为他是按照任务的完成顺序往队列里放，所以我们可以全部提交后，利用他的poll或者take方法遍历任务，先完成的任务返回就可以直接消费。

**Future**

讲到这里我觉得有必要提一下Future，因为线程池中投递任务submit方法均为返回Future这个对象。Future你可以把它理解成对这个任务的建模，你得这个对象可以利用这个对象来管理任务的生命周期，例如get方法获取结果，cancel来取消这个任务，以及isDnoe来判断任务是否取消等。api没有什么难理解的地方，主要是取消任务这一块需要结合中断来理解，cancel参数的Boolean值就是说能不能给这个任务发中断，如果可以他内部实际就是通过中断来停止任务，需要用户代码响应中断。FutureTask中的cancel源码如下：

```java
if (mayInterruptIfRunning) {
    try {
        Thread t = runner;
        if (t != null)
            t.interrupt();
    } finally { // final state
        UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
    }
}
```


## 二、CompletableFuture（重要）

上面简单说了下Future，Future是jdk5.0就已经引进的，但是他的能力非常的弱，主要是缺少了一个回调的机制，很多框架都基于它提供了增强版像guava的<code>ListenableFuture</code>和spring中的<code>ListenableFuture</code>。直到java8出现了CompletableFuture才弥补了jdk的这个特性。    
可能很多人没有注意到这个类，因为平时没关注这方面，其实如果好好的学习下这个类就会发现这个类的功能非常强大，和stream类似的设计思想，使用非常简洁。可以基于教程好好研究一下，这里介绍下常用的操作。        

在以前我们投递到线程中任务返回的Future中，我们只能实现一些简单的轮询，取消等api。如果现在有这样的一些类似的需求：    

**执行一个任务，当任务执行完的时候执行一个动作（相当于任务执行完触发回调）**

```java
    CompletableFuture.runAsync(() -> System.out.println("hello word")).whenComplete((aVoid, throwable) -> System.out.println("任务完成"));
```

**任务执行完的时候在发起另外一个任务（这里是有顺序性的，第二个依赖于第一个任务）**

```java
CompletableFuture.supplyAsync(() -> 12).thenApply(Function.identity()).thenAccept(System.out::println);
```

**同时执行多个任务，当全部完成的时候执行一个动作**
```java
Integer join = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 1;
        }).thenCombine(CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 2;
        }), Integer::sum).join();
```


可以看到上面的这些任务都不简单，但是如果使用CompletionStage却变得十分容易，对任务的组合正是CompletableFuture的强项，下面大概说下CompletionStage的api。   
通过观察就会发现CompletionStage的一种类型操作都会有三种重载形式，分别如下：   
1. XXX()
2. XXXAsync()
3. XXXAsync(executors参数)

第一个表示这个操作在当前线程池中的当前线程直接执行，第二个表示会重新投递到当前线程池执行，第三个则表示会重新投递到传入的线程池继续执行。一般情况都会采用第一种形式和第三种，第一种是最常用的，可以减少线程的上下文切换，第三种情况主要用户切换线程池，我们很多时候会根据不同的任务比如io密集型，cpu密集型创建不同的线程池，考虑这样一种情况，我创建了一个ncpu+1的线程池1，和一个上限500的线程池2，第一个任务是cpu密集型，第二个依赖于第一个且是io密集型，这时候我们可以选择将第一个投递到线程池1，然后第二个通过传入线程池参数重新投递到线程池2中执行。

**主要方法：**     
1. thenRun   前一个任务<font color="red">正常执行</font>完后执行一个任务
2. thenAccept
3. thenApply
4. thenCompose 
5. handle   *前一个任务执行完后执行(包括前一个抛出异常)，如果前面任务未抛出异常当前任务抛出异常则结果就是抛出异常，如果前面任务抛出异常e1当前任务抛出异常e2则结果就是抛出异常e1*
6. whenComplete  
7. exceptionally  如果前一个任务未抛出异常则不执行
8. XXXBoth(thenCombine)   两个任务全都<font color="red">正常完成</font>时执行
9. XXXEither  当最先完成的任务是<font color="red">正常完成</font>时执行

上面的方法大致可以再分为三组，分别是1-4,5-7,8-9。       

1-4： 这个很好理解，四个方法都十分类似，这里要提下的就是thenApply和thenCompose，会感觉有点难理解，你可以从java的stream中map和flatMap的角度来理解，这两个方法做的事实际是一样的，只不过形式不一样，主要也是在和stream结合的过程中会用到，所以一般很少用到thenCompose。另外有一点很重要：*如果前一个任务执行过程抛出了异常那么这个任务就不会执行也**不会有任何提示**，除非你调用CompletableFuture的get等获取结果的方法会再次得到异常，不然这个异常信息就丢了，需要十分注意这个点。*   
5-7： 1-4的方法如果前面的任务抛出异常则会导致1-4任务的不执行。5-7的方法都可以对异常进行处理，如果前一个抛出了异常，会有参数传入，可以做相应的处理，很多时候可以利用这三个方法来记录日志，你看whenComplete就完全是一个透传的效果    
8-9： 这两组类型分别是对两个任务的与和或条件的组合工具方法。没什么难理解的，主要就是thenCombine，其实他实现的语义就是applyAfterBoth，只不过名字稍微不同而已。另外说下XXXEither，他的语义是任意一个任务执行完成后执行相应的动作，需要注意的地方就是如果最先完成的那个任务抛出的是异常这个任务就不会执行。

再次提醒下异常，如果你对最后的CompletablFuture会调用get等取结果的方法那没什么，执行过程中抛出的异常会再次抛出，但是如果你只是调用后不再去取结果，就像thenApply结尾那么就一定要非常小心，如果前一个方法抛出异常你的thenApply的任务便不会执行，而且都没有什么提示。你可以对相应的任务包装打印异常在rethrow,如下所示：

```java
public class AsyncableWrapper {

    public static final Runnable of(Runnable runnable) {
        return () -> {
            try {
                runnable.run();
            } catch (Exception e) {
                log.error("执行异步任务失败", e);
                throw e;
            }
        };
    }


    public <T> Consumer<T> ofConsumer(Consumer<T> consumer) {
        return o -> {
            try {
                consumer.accept(o);
            } catch (Exception e) {
                log.error("执行异步任务失败，", e);
                throw e;
            }
        };
    }

    public <T> Supplier<T> ofSupply(Supplier<T> supplier) {
        return () -> {
            try {
                return supplier.get();
            } catch (Exception e) {
                log.error("执行异步任务失败，", e);
                throw e;
            }
        };
    }
}
```

下面的这个代码你可以试试执行的结果，用来测试异常：
```java
CompletableFuture.supplyAsync(() -> 1).whenComplete((o, throwable) -> {
    System.out.println("when1");
    throw new RuntimeException();
}).thenAccept(o -> {
    System.out.println("accetp2");
}).thenApply(aVoid -> {
    System.out.println("apply");
    return 1;
}).whenComplete((integer, throwable) -> {
    System.out.println("when2");
}).thenAccept(integer -> {
    System.out.println("accetp");
});
```
**获取结果**     

join: 在之前Future的api中，只有get方法，会抛出受检查异常，受检查在lambda表达式中需要捕获使得代码看上去不那么美观，因此引入了join，除了包装了受检异常，其他行为和get一样    

getNow:可以立即返回，参数可以写入默认值，在轮询的场景中会有用到     


**工具方法**    

anyOf:用于等待一组任务任意一个最先完成   

allOf:用于等待一组任务全部完成    

这两个方法和上面的XXXEither和XXXBoth很相似，只不过是多个CompletablFuture。抛出异常的规则也是一样的。常用形式如下：     

<code>CompletableFuture.allOf(c1, c2, c3).join();</code>    

## 三、stream中的parallel（并行流）     

在处理一批任务的时候，大部分场景都会有个集合，例如一个id列表，然后我们需要获取每个id的信息，通过http接口，但是没有批量接口，这时候我们可以采用并行来提高性能。     

流中有个简单的方法parallel可以并行执行，如下所示：

```java
Arrays.stream(new int[]{1, 2, 3}).parallel().forEach(operand -> {
    try {
        System.out.println("执行任务:" + operand);
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});
```

上面的转换成并行流十分简单，执行时间一秒左右，就多了一个方法调用。那么如果每个任务执行完还有第二个步骤那怎么办呢，很容易想到结合ComplablFuture使用，那就是如下的形式：    

```java
CompletableFuture[] completableFutures = Arrays.stream(new Integer[]{1, 2, 3})
        .map(operand -> CompletableFuture.supplyAsync(() -> {
            try {
                System.out.println("执行任务:" + operand);
                Thread.sleep(1000);
                return operand;
            } catch (InterruptedException e) {
                e.printStackTrace();
                return 0;
            }
        })).map(integerCompletableFuture -> integerCompletableFuture.thenApply(integer -> {
            try {
                Thread.sleep(1000);
            } catch (Exception e) {
            }
            return integer;
        })).toArray(CompletableFuture[]::new);
CompletableFuture.allOf(completableFutures).join();
```

上面的耗时大概在2秒左右，Stream和ComplablFuture组合功能是十分强大的，另外你可以注意到上面的代码中我移除了parallel()方法，因为ComplablFuture本身就利用了线程池，再利用parallel()是没必要的，反而会增加线程上下文的切换。     

**实际执行的线程池**      

在使用并行流和异步的过程中，肯定会非常好奇到底实际执行代码是在哪里，异步可能会好理解些，因为他很多方法例如thenAcceptAsync提供了线程池是你可以配置的。但是如果不传的那么实际使用的是ForkJoinPool。代码如下：

```java
  private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

并行流内部使用的也是上述的线程池，但是并行流却没有提供显示设置线程池的方法，这就导致有些阻塞的方法不适合采用并行流，其实他也是可以设定线程池的只不过不是像你想的那样，代码如下：     

```java
ExecutorService executorService = Executors.newWorkStealingPool();
executorService.execute(() -> {
        CompletableFuture.supplyAsync(() -> {
            System.out.println(1);
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            throw new RuntimeException();
        }).runAfterEither(CompletableFuture.supplyAsync(() -> {
            System.out.println(2);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 1;
        }), () -> System.out.println(123));
});
```

关键点就在于上面代码中的executorService，他是ForkJoinPool的实例，**并行流执行的过程中如果发现当前线程是ForkJoinPool的实例，那么会利用当前的ForkJoinPool来并行执行**，从而改变了线程池。如果只是计算操作，没有涉及io和锁等阻塞那么使用默认的线程池是很不错的行为，就像平时对集合使用stream来计算就完全没必要改变线程池。但是在使用线程池提高性能的很多时候都会涉及io操作，如Rpc，Db，Http等操作，这时候完全有必要根据相应的业务场景提供一个合适的线程池，而不是使用一个统一的线程池。关于利用并行stream还是ComplablFuture，如果不涉及io以及任务组合等操作，我更会倾向使用stream，更多的情况下我会选择使用ComplablFuture，结构更清晰。    

关于线程池的总结可以参考我的这篇文章 [java线程池和中断总结](https://www.cnblogs.com/chenfangzhi/p/9912484.html)

## 四、实际使用的另外一点总结：     

刚开始接触异步的时候觉得他是提升性能的银弹，但是其实很多技术都有适合的场景，不能为了技术而技术。这里举个反例，当时在公司调用rpc的时候例如根据id列表批量获取信息，因为不想麻烦别人有刚好试试异步api，就采用了每个id调用一次，利用异步来降低延迟的方案，后来实际证明是非常错误的！假设有100个id，如果对方提供了批量操作的rpc，那么一次往返即可，采用异步方案多增加199次调用，吞吐量严重降低，另外因为接口有调用限制，并发上去后接口的全部返回失败！这种场景rpc每个操作耗时短，就非常适合提供批量操作而不是批量。    
再来个正面例子，公司最近对接腾讯云的人脸识别服务，因为是Http接口，而且每个接口返回比较慢，所以非常适合采用线程池和异步来优化延迟。

