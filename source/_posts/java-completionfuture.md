---
title: Java8 之 CompletableFuture
date: 2017年10月29日21:19:59
thumbnail: http://www.runoob.com/wp-content/uploads/2013/12/java.jpg
tags: 
 - CompletableFuture
 - JDK
 - Java8
categories: 
 - Java
---

**使用CompletableFuture的有以下好处：**

1. 使用CompletableFuture类提供的特性可以创建异步任务、为客户提供异步API。
2. 使用CompletableFuture可以以改善程序的性能，加快程序的响应速度。尤其是在执行比较耗时的操作时，例如调用一个或多个远程服务。
3. CompletableFuture类还提供了异常管理的机制，让你有机会抛出/管理异步任务执行中发生的异常。
4. 将同步API的调用封装到一个CompletableFuture中，你能够以异步的方式使用其结果。
5. 如果异步任务之间相互独立，或者它们之间某一些的结果是另一些的输入，你可以将这些异步任务构造或者合并成一个。
6. 你可以为CompletableFuture注册一个回调函数，在Future执行完毕或者它们计算的结果可用时，针对性地执行一些程序。
7. 你可以决定在任意时候结束程序的运行，是等待由CompletableFuture对象构成的列表中所有的对象都执行完毕，还是只要其中任何一个首先完成就中止程序的运行。

<!--more-->
**由于CompletableFuture**类实现了**Future**和**CompletionStage**接口，下面我们先来看下两个接口分别提供了什么功能：

----------
# **Future**

Future接口在JDK1.5中被引入，用来描述一个异步计算的结果，但是获取一个结果的方法较少，在结果没有真正返回前，`future.get()`方法会阻塞住调用线程，要么通过轮询`future.isDone()`，确认完成后，再调用get()获取值。


```java
    ExecutorService executorService = Executors.newSingleThreadExecutor();
    Future<String> future = executorService.submit(new Callable<String>() {
        @Override
        public String call() throws Exception {
            TimeUtils.sleep5Seconds();
            return "I am finished.";
        }
    });
    
    System.out.println("main thread -> I am not blocked");
    try {
        System.out.println("-----> " + future.get());
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();//如果线程内出现异常，则会catch到ExecutionException
    }
    System.out.println("main thread -> I am blocked");
    
    //注意：ExecutorService使用的线程默认是非守护线程，所以执行完毕后，测试代码不会停止，要让其结束掉，有两种方式：1，设为守护线程 2，显式调用其shutdown()方法
    //ExecutorService executorService = Executors.newSingleThreadExecutor((r) -> {
    //    Thread thread = new Thread(r);
    //    thread.setDaemon(true);
    //    return thread;
    //});
    executorService.shutdown();//本例采用方法2
```
输出结果为：
```java
main thread -> I am not blocked
----->  This is the result after async process
main thread -> I am blocked
```
或者通过`future.isDone()`方法来判断future是否完成，然后再调用`future.get()`方法就能直接拿到结果了。这里只是介绍下`future.isDone()`方法，其实两者区别不大，仅仅是使用`future.isDone()`的时候你还可以处理下其他事情，使用`future.get()`就完全阻塞死了。做法：在上述代码第10行后插入以下代码。
```java
    while (!future.isDone()) { 
        TimeUtils.sleep(10);//如果还没有完成，则休眠10ms
    }
```
如果future执行的时间很长，那么`future.get()`方法会一直阻塞下去，这时候，在一些使用场景下可能会出现问题，所以你可以能需要使用 `future.get(long timeout, TimeUnit)`方法，在给定的时间拿不到结果，则抛出一个TimeoutException
```java
    try {
        System.out.println(future.get(2, TimeUnit.SECONDS));
    } catch (InterruptedException | ExecutionException | TimeoutException e) {
        e.printStackTrace();
    }
```
可以看到future并没有我们意料的好用，我们并不能通过future来达成以下目标：

 - 在多个Future组成的集合中，等待所有Future中的任务都完成
 - 在多个Future组成的集合中，仅等待最快结束的任务完成（有可能因为它们试图通过不同的方式计算同一个值），并返回它的结果
 - 将两个异步计算合并为一个（这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果）
 - 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）
 - 应对Future的完成事件（即当Future的完成事件发生时会收到通知，并能使用Future计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）

**那么下面，我们看看CompletionStage类是如何将这些都变为可能。**

----------

# **CompletionStage**
CompletionStage是一个计算阶段（可能同步、也可能是异步），它可以在另外一个CompletionStage完成后执行一个动作(Action)或者去获取一个值，然后还可以去触发其他的CompletionStage。更多API可以参考[CompletionStage Docs](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)

 1. CompletionStage本身可以进行组成、复合（compose、combine）操作。
 2. CompletionStage产生的结果可以再进行Function、Consumer、Runnable操作，使用哪个函数取决于入参和出参。例如`stage.thenApply(x -> square(x)).thenAccept(x -> System.out.print(x)).thenRun(() -> System.out.println())`
 3. 一个CompletionStage的执行步骤可以由一个CompletionStage触发，或者另外两个全部完成触发，或者另外两个其中一个完成触发。例如分别对应于run、runAfterBoth、runAfterEither方法。当时用either方法时，并不能保证是哪个结果导致的下一个stage的执行。
 4. CompletionStage可以同步或者异步（Async）执行。并且提供了重载的方法，你可以使用内置的Executor或者自己提供一个。
 5. 关于 whenComplete和handle ***//TODO***

由于CompletionStage是接口，其具体实现在CompletionFuture里，我们下面就看看CompletionStage如何应用上面的这些功能

----------

# **CompletionFuture**  
[更多API可以参考](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
## 1，基本用法 
### 1.1.1 使用new的方式
启动一个线程，去获取结果。当线程执行完毕调用`future.complete(T value)`方法，此时`future.whenComplete`会收到回调.
```java
    CompletableFuture<Double> future = new CompletableFuture<>();
    new Thread(() -> future.complete(SomeService.get())).start();
    System.out.println("main thread -> I am not blocked");
    future.whenComplete((v, t) -> {
        Optional.ofNullable(v).ifPresent(System.out::println);
        Optional.ofNullable(t).ifPresent(Throwable::printStackTrace);
    });
    System.out.println("main thread -> I am not blocked");
```
其中SomeService.get()是一个模拟的耗时的任务，其作用是5秒钟后返回一个随机数
```java
    static double get() {
        Random random = new Random(System.currentTimeMillis());
        TimeUtils.sleep5Seconds();
        return random.nextDouble();
    }
```
执行结果
```java
main thread -> I am not blocked
main thread -> I am not blocked
0.7731229511499652
```
### 1.1.2 使用supplyAsync + pipleline的形式
```java
    CompletableFuture.supplyAsync(SomeService::get)
            .whenComplete((v, t) -> System.out.println("whenComplete value is -> " + v))
            .thenCompose(i -> CompletableFuture.supplyAsync(() -> i + 10))//模拟 将结果加10处理
            .thenAccept((v) -> System.out.println("thenAccept value is -> " + v))
            .thenRun(() -> System.out.println("thenRun -> do some end task"));
```
执行结果为：
```java
whenComplete value is -> 0.5416761173806217
thenAccept value is -> 10.541676117380621
thenRun -> do some end task
```
### 1.1.3 并行Stream + CompletionFuture结合使用。
这里将4个整数分别乘以获取到的随机数，并统计时间：
```java
    long start = System.currentTimeMillis();
    Stream.of(1, 2, 3, 4)
            .map(d -> CompletableFuture.supplyAsync(() -> d * SomeService.get()))
            .parallel()
            .map(CompletableFuture::join).forEach(System.out::println);
    System.out.println("time costs -> " + (System.currentTimeMillis() - start) + "ms");
```
执行结果为：
```java
1.6149546280106335
3.2281176801818123
0.8070294200454531
2.42243194201595
time costs -> 5058ms
```
可以看到获取一次随机数需要5秒，总共也只需要5秒多点。
## 2，CompletableFuture API详解
函数名 |API    | 功能                      
----   |----   | ------      
join|`join()`  | 本stage完成后，获取其结果    
whenComplete|`CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);`  | 本stage正常完成后或出现了异常，都会进入该方法，注意函数是BiConsumer，无需返回值   
exceptionally|`CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);`  | 本stage出现了异常，会进入本方法，可以对异常处理再返回新的CompletionStage       
handle|`<U> CompletableFuture<U> handle(BiFunction<? super T,Throwable,? extends U> fn)`  | handle相当于whenComplete和exceptionally的加强版，可操作空间大
thenApply|`<U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn)`  | 本stage完成后，执行一个Function，用其结果构建一个新的stage并返回           
thenAccept|`CompletionStage<Void> thenAccept(Consumer<? super T> action)`   | 本stage完成后，执行一Consumer，用Void构建一个新的stage并返回
thenRun|`CompletionStage<Void> thenRun(Runnable action)`   | 本stage完成后，执行一个Runnable，用Void构建一个新的stage并返回
thenCombine|`<U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn)`| 本stage和给定的stage都正常完成后，执行一个BiFunction，用其结果构建一个新的stage并返回
thenCompose|`<U> CompletableFuture<U> thenCompose(Function<? super T,? extends CompletionStage<U>> fn)`   | 本stage正常完成后，用其结果构造一个新的stage，和thenApply类似，只不过这个需要自己构造stage，thenApply会自动构造
thenAcceptBoth|`<U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action)` | 本stage和other stage都完成后，再执行action，其他带有Both的类似
runAfterEither|`CompletableFuture<Void> runAfterEither(CompletionStage<?> other, Runnable action)`   | 本stage和other stage有一个完成后，就执行action，其他带有Either的类似
anyOf|`static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)`   | 本stage和other stage有一个完成后，就执行action，其他带有Either的类似
allOf|`static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)`   | 本stage和other stage有一个完成后，就执行action，其他带有Either的类似

示例代码：
```java
    @Test
    public void whenCompleteAndExceptionally() {
        String result = CompletableFuture
                .supplyAsync(() -> {
                    int i = 3 / 0;
                    return "Okay";
                })
                .whenComplete((v, t) -> {
                    Optional.ofNullable(v).ifPresent(System.out::println);
                    System.out.println("whenComplete t = " + t);
                })
                .exceptionally(t -> {
                    System.out.println("exceptionally t = " + t);
                    return "Okay after exception";
                })
                .join();
        System.out.println(result);// 结果 Okay after exception
        // 注意whenComplete和exceptionally都会打印
    }
    // 执行完毕一个CompletableFuture，再对其进行下一个CompletableFuture操作
    @Test
    public void thenCompose() {
        CompletableFuture.supplyAsync(SomeService::get)
                .thenCompose(i -> CompletableFuture.supplyAsync(() -> 10 * i))
                .thenAccept(System.out::println);
                
        TimeUtils.sleep5Seconds();//休眠以等待CompletableFuture执行完，下同
    }

    // 执行完毕一个CompletableFuture，再执行另外一个CompletableFuture，最后在对两个的执行结果执行BiFunction
    @Test
    public void testThenCombine() {
        CompletableFuture.supplyAsync(SomeService::get)
                .thenCombine(CompletableFuture.supplyAsync(() -> 2.0d), (r1, r2) -> r1 + r2)
                .thenAccept(System.out::println);

        TimeUtils.sleep5Seconds();
    }
    @Test
    public void thenApply() {
        String result = CompletableFuture.supplyAsync(() -> "hello").thenApply(s -> s + " world").join();
        System.out.println(result);//结果 hello world
    }
    // thenAcceptBoth 需要两个都正常执行完毕，才能得到结果
    @Test
    public void testThenAcceptBoth() {
        CompletableFuture.supplyAsync(SomeService::get)
                .thenAcceptBoth(CompletableFuture.supplyAsync(() -> 2.0D), (r1, r2) -> System.out.println(r1 + r2));

        TimeUtils.sleep5Seconds();
    }

    @Test
    public void anyOf() {
        List<CompletableFuture<Double>> collect = Stream.of(1, 2, 3, 4)
                .map(i -> CompletableFuture.supplyAsync(SomeService::get))
                .collect(Collectors.toList());
        CompletableFuture.anyOf(collect.toArray(new CompletableFuture[collect.size()]))
                .thenAccept(System.out::println) //结果 0.3960857305756089
                
        TimeUtils.sleep5Seconds();
    }
```

## 3，CompletableFuture中的线程池
一般对于CompletableFuture会有多个重载的方法，例如以下：
```java
    public CompletableFuture<Void> thenRun(Runnable action)
    public CompletableFuture<Void> thenRunAsync(Runnable action)
    public CompletableFuture<Void> thenRunAsync(Runnable action,  Executor executor)
```
那么这三个方法中的代码，分别会处于什么线程中执行呢？（这里只以thenRun为例，其他的方法类似）测试代码如下：
```java
    ExecutorService executor = Executors.newCachedThreadPool();
    CompletableFuture.supplyAsync(SomeService::get, executor)
            .thenRun(() -> System.out.println("thenRun1  ->" + Thread.currentThread()))
            .thenRunAsync(() -> System.out.println("thenRunAsync  ->" + Thread.currentThread()))
            .thenRun(() -> System.out.println("thenRun2  ->" + Thread.currentThread()))
            .thenRunAsync(() -> System.out.println("thenRunAsync in executor  ->" + Thread.currentThread()), executor)
            .thenRun(() -> System.out.println("thenRun3  ->" + Thread.currentThread()))
```
可以看到如下结果（为了对比，格式上对输出结果加以调整，顺序未变）：
```
SomeService get  ->Thread[pool-1-thread-1,5,main]
thenRun1         ->Thread[pool-1-thread-1,5,main]
thenRunAsync  ->Thread[ForkJoinPool.commonPool-worker-1,5,main]
thenRun2      ->Thread[ForkJoinPool.commonPool-worker-1,5,main]
thenRunAsync in executor  ->Thread[pool-1-thread-1,5,main]
thenRun3                  ->Thread[pool-1-thread-1,5,main]
```
`thenRun(Runnable action)` 是在**上一步骤中的的执行线程中执行**
`thenRunAsync(Runnable action)` ***一般***是在JDK为提供的默认线程池ForkJoinPool.commonPool()中执行，具体是和CPU核数、JVM配置有关，这里不在多说，可以简单参考：[ForkJoinPool的commonPool相关参数配置](http://www.jianshu.com/p/1b5f4ea0074a)
`thenRunAsync(Runnable action,  Executor executor)` 是在提供的线程池中执行

## 4，不要在whenComplete方法中里面产生异常
如下代码
```java
        CompletableFuture.supplyAsync(SomeService::get, executor)
                .whenComplete((v, t) -> {
                    System.out.println("whenComplete -> ");
                    System.out.println("v = " + v);
                    throw new RuntimeException();
                })
                .thenAccept((v) -> System.out.println("thenAccept  -> " + v))
                .thenRun(() -> System.out.println("thenRun" ))
                .thenApply((v) -> {
                    System.out.println("thenApply  -> " + v);
                    return 1;
                });
```
执行结果如下
```java
whenComplete
v = 0.43074339929275096
```
可以看到后续代码都无法执行。

# **进阶**
## 1，代码示例
### 1.1.1 如何能让多个CompletableFuture中的一个失败，则认为整体失败。

我们可能会遇到这样的业务逻辑（例如校验），我们需要用多个CompletableFuture去异步并发做不同的工作，如果其中一个CompletableFuture失败，则直接返回失败。
我们知道`CompletableFuture.anyOf()`适用于其中一个成功则成功的场景，与我们现在的需求刚好相反。而`CompletableFuture.allOf()`则是全部成功才成功，虽然和我们这个场景有点类似，但是有一点不同：如果有某个CompletableFuture失败的时候，则仍需要等待其他所有CompletableFuture都执行完才会结束，我们认为它**失败的不够快**。使用`CompletableFuture.allOf()`的实例代码如下：

```java
    private static CompletableFuture<?> composed(CompletableFuture<?> ... futures) {
        return CompletableFuture.allOf(futures);
    }
    
    @Test
    public void testCompletableFutureFailFast() {
        LocalTime now = LocalTime.now();
        CompletableFuture<String> checkFuture1 = CompletableFuture.supplyAsync(CheckService1::check);
        CompletableFuture<String> checkFuture2 = CompletableFuture.supplyAsync(CheckService2::check);
        
        composed(checkFuture1, checkFuture2)
               . whenComplete((v, t) -> {
                    System.out.println("v = " + v);
                    System.out.println("t = " + t);
                    System.out.println("TimeCosts:" + Duration.between(now, LocalTime.now()).getSeconds());
                });
        TimeUtils.sleep5Seconds();// 睡眠让Test执行完
    }
    private static class CheckService1 {//模拟一个校验，需要耗时1s，校验会失败
        static String check() {
            TimeUtils.sleep1Seconds();
            throw new RuntimeException("CheckService1 check fail");
        }
    }
    private static class CheckService2 {//模拟一个校验，需要耗时3s
        static String check() {
            TimeUtils.sleep3Seconds();
            return "OKAY";
        }
    }
```
执行结果如下：
```java
v = null
t = java.util.concurrent.CompletionException: java.lang.RuntimeException: CheckService1 check fail
TimeCosts:3
```
可以看到，即使CheckService1校验失败（耗时1s），也会等待CheckService2校验完（耗时3s），所以这种做法**失败的不够快**！

此时，对代码进行如下修改,结合`CompletableFuture.anyOf()`和`CompletableFuture.allOf()`两个函数
```java
    private static CompletableFuture<?> composed(CompletableFuture<?> ... futures) {
        // 首先构造一个当全部成功则成功的CompletableFuture
        CompletableFuture<?> allComplete = CompletableFuture.allOf(futures);
        
        // 再构造一个当有一个失败则失败的的CompletableFuture
        CompletableFuture<?> anyException = new CompletableFuture<>();
        for (CompletableFuture<?> completableFuture : futures) {
            completableFuture.exceptionally((t) -> {
                //对于传入的futures列表，如果一个有异常，则把新建的CompletableFuture置为成功
                anyException.completeExceptionally(t);
                return null;
            });
        }
        // 让allComplete和anyException其中有一个完成则完成
        // 如果allComplete有一个异常，anyException会成功完成，则整个就提前完成了
        return CompletableFuture.anyOf(allComplete, anyException);
    }
    // 其他代码不变
```
重新执行结果如下：
```java
v = null
t = java.util.concurrent.CompletionException: java.lang.RuntimeException: CheckService1 check fail
TimeCosts:1
```
可以看到，第一个CheckService1校验失败的时候则整体失败了，**失败的已经够快了**！


## **参考**
- [Java8实战](https://book.douban.com/subject/26772632/)
- [ForkJoinPool的commonPool相关参数配置](http://www.jianshu.com/p/1b5f4ea0074a)
- [CompletableFuture — Aggregate Future to Fail Fast](https://stackoverflow.com/questions/33783561/completablefuture-aggregate-future-to-fail-fast)