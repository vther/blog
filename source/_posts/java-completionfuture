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

>  1. 使用CompletableFuture类提供的特性可以创建异步任务、为客户提供异步API。
>  2. 使用CompletableFuture可以以改善程序的性能，加快程序的响应速度。尤其是在执行比较耗时的操作时，例如调用一个或多个远程服务。
>  3. CompletableFuture类还提供了异常管理的机制，让你有机会抛出/管理异步任务执行中发生的异常。
>  4. 将同步API的调用封装到一个CompletableFuture中，你能够以异步的方式使用其结果。
>  5. 如果异步任务之间相互独立，或者它们之间某一些的结果是另一些的输入，你可以将这些异步任务构造或者合并成一个。
>  6. 你可以为CompletableFuture注册一个回调函数，在Future执行完毕或者它们计算的结果可用时，针对性地执行一些程序。
>  7. 你可以决定在任意时候结束程序的运行，是等待由CompletableFuture对象构成的列表中所有的对象都执行完毕，还是只要其中任何一个首先完成就中止程序的运行。

<!--more-->
**由于CompletableFuture**类实现了**Future**和**CompletionStage**接口，下面我们先来看下两个接口分别提供了什么功能：

----------
# **Future**

> Future接口在JDK1.5中被引入，用来描述一个异步计算的结果，但是获取一个结果的方法较少，在结果没有真正返回前，`future.get()`方法会阻塞住调用线程，要么通过轮询`future.isDone()`，确认完成后，再调用get()获取值。


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
或者可以在上述代码第10行后插入以下代码，通过`future.isDone()`方法来判断future是否完成，然后`future.get()`方法就能直接拿到结果了。这里只是介绍下`future.isDone()`方法，其实两者并没有太大区别。
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

 - 等待Future集合中的所有任务都完成。  
 - 仅等待Future集合中最快结束的任务完成（有可能因为它们试图通过不同的方式计算同一个值），并返回它的结果。
 - 将两个异步计算合并为一个——这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果。
 - 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）。
 - 应对Future的完成事件（即当Future的完成事件发生时会收到通知，并能使用Future计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）

**那么下面，我们看看CompletionStage类是如何将这些都变为可能。**

----------

# **CompletionStage**
CompletionStage是一个计算阶段（可能同步、也可能是异步），它可以在另外一个CompletionStage完成后执行一个动作(Action)或者去获取一个值，然后还可以去触发其他的CompletionStage。更多API可以参考[CompletionStage Docs](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)

> CompletionStage本身可以进行compose操作。

> CompletionStage产生的结果可以再进行Function、Consumer、Runnable操作，使用哪个函数取决于入参和出参。例如`stage.thenApply(x -> square(x)).thenAccept(x -> System.out.print(x)).thenRun(() -> System.out.println())`

> 一个CompletionStage的执行步骤可以由一个CompletionStage触发，或者另外两个全部完成触发，或者另外两个其中一个完成触发。例如分别对应于run、runAfterBoth、runAfterEither方法。当时用either方法时，并不能保证是哪个结果导致的下一个stage的执行。

> CompletionStage可以同步或者异步（Async）执行。并且提供了重载的方法，你可以使用内置的Executor或者自己提供一个。

> whenComplete  handle 
> 

由于CompletionStage是接口，其具体实现在CompletionFuture里，我们下面就看看CompletionStage如何应用上面的这些功能

----------

# **CompletionFuture**
## 1，基本用法
1.1 使用new的方式，启动一个线程，去获取结果。当线程执行完毕调用`future.complete(T value)`方法，此时`future.whenComplete`会收到回调.
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
1.2 使用supplyAsync方式，使用pipleline的形式。
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
1.3 并行Stream和CompletionFuture结合使用。这里将4个整数分别乘以获取到的随机数，并统计时间：
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
## 2，map和flatMap的区别
***map***：如果值存在，则执行提供的Function
```java
    public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
    }
```
例如：从一堆未处理的文件列表中，找到第一个未处理过的文件，并得到其输入流
```java
    Optional<FileInputStream> fis =
               names.stream()
                    .filter(name -> !isProcessedYet(name))
                    .findFirst()
                    .map(name -> new FileInputStream(name));
```

                            
***flatMap***：用法和map类似，但是参数需要提供使用Optional包裹的Function，调用后，返回的也是有Optional包裹的结果。简单而言就是：***map需要的Function是T到U，而flatMap是T到Optional<U>***
                            
```java
    public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Objects.requireNonNull(mapper.apply(value));
        }
    }
```
## 3，orElse、orElse、orElseThrow
***orElse***：如果值不存在，则返回一个默认值
```java
    public T orElse(T other) {
        return value != null ? value : other;
    }
```
***orElseGet***：如果值不存在，则通过一个Supplier提供默认值
                            
```java
    public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }
```
***orElseThrow***：如果值不存在，则抛出异常
                            
```java
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }
```
例如：通过订单ID查询订单，查询不到则抛出异常OrderNotExistsException
```java
    public Order getOrder(String orderId) {
        return Optional.ofNullable(orderId)
                .map(orderReposity::findByOrderId)
                .orElseThrow(OrderNotExistsException::new);
    }
```

## 4，范例

从配置文件中读取整数配置项

```java
    // 以前用法
    public static int getIntProp(Properties props, String name) {
        String value = props.getProperty(name);
        if (value != null) {
            try {
                int i = Integer.parseInt(value);
                if (i > 0) {
                    return i;
                }
            } catch (NumberFormatException nfe) { }
        }
        return 0;
    }
    // 使用Optional
    public static int getIntPropWithOptional(Properties props, String name) {
        return Optional.ofNullable(props.getProperty(name))
                .flatMap(Main::string2int)
                .filter(i -> i > 0).orElse(0);
    }
    // String转整数
    public static Optional<Integer> string2int(String s) {
        try {
            return Optional.of(Integer.parseInt(s));
        } catch (NumberFormatException e) {
            return Optional.empty();
        }
    }
```

## **参考**

- [Java 8 Optional类深度解析](http://www.importnew.com/6675.html)
