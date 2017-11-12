---
title: Spring 之 重试框架[spring-retry]
date: 2017年11月12日21:19:59
thumbnail: https://spring.io/img/homepage/icon-spring-framework.svg
tags: 
 - Spring
 - spring-retry
 - 重试
 - 熔断
categories: 
 - Spring
---

spring-retry项目实现了重试和熔断功能，目前已用于SpringBatch、Spring Integration等项目。

RetryOperations定义了重试的API；
RetryTemplate提供了线程安全的模板实现，同于Spring 一贯的API风格，RetryTemplate将重试、熔断功能封装到模板中，提供健壮和不易出错的API供大家使用。
RecoveryCallback定义了重试操作；
RetryState用于定义有状态的重试。

<!--more-->

首先，RetryOperations接口API：
```java
public interface RetryOperations {
   <T, E extends Throwable>T execute(RetryCallback<T, E>retryCallback) throws E;
   <T, E extends Throwable>T execute(RetryCallback<T, E>retryCallback, RecoveryCallback<T> recoveryCallback) throws E;
   <T, E extends Throwable>T execute(RetryCallback<T, E>retryCallback, RetryState retryState) throws E, ExhaustedRetryException;
   <T, E extends Throwable>T execute(RetryCallback<T, E>retryCallback, RecoveryCallback<T> recoveryCallback, RetryStateretryState)
         throws E;
}
```
通过RetryCallback定义需重试的业务服务，当重试超过最大重试时间或最大重试次数后可以调用RecoveryCallback进行恢复，比如返回假数据或托底数据。


# **RetryPolicy**
那什么时候需重试？spring-retry是当抛出相关异常后执行重试策略，定义重试策略时需要定义需重试的异常（如因远程调用失败的可以重试、而因入参校对失败不应该重试）。只读操作可以重试，幂等写操作可以重试，但是非幂等写操作不能重试，重试可能导致脏写，或产生重复数据。
重试策略有哪些呢？spring-retry提供了如下重试策略(RetryPolicy):
 - NeverRetryPolicy：只允许调用RetryCallback一次，不允许重试；
 - AlwaysRetryPolicy：允许无限重试，直到成功，此方式逻辑不当会导致死循环；
 - SimpleRetryPolicy：固定次数重试策略，默认重试最大次数为3次，RetryTemplate默认使用的策略；
 - TimeoutRetryPolicy：超时时间重试策略，默认超时时间为1秒，在指定的超时时间内允许重试；
 - CircuitBreakerRetryPolicy：有熔断功能的重试策略，需设置3个参数openTimeout、resetTimeout和delegate，稍后详细介绍该策略；
 - CompositeRetryPolicy：组合重试策略，有两种组合方式，乐观组合重试策略是指只要有一个策略允许重试即可以，悲观组合重试策略是指只要有一个策略不允许重试即可以，但不管哪种组合方式，组合中的每一个策略都会执行。
 
# **BackOffPolicy**
重试时的退避策略是什么？是立即重试还是等待一段时间后重试，比如是网络错误，立即重试将导致立即失败，最好等待一小段时间后重试，还要防止很多服务同时重试导致DDos。
BackOffPolicy提供了如下策略实现：
 - NoBackOffPolicy：无退避算法策略，即当重试时是立即重试；
 - FixedBackOffPolicy：固定时间的退避策略，需设置参数sleeper和backOffPeriod，sleeper指定等待策略，默认是Thread.sleep，即线程休眠，backOffPeriod指定休眠时间，默认1秒；
 - UniformRandomBackOffPolicy：随机时间退避策略，需设置sleeper、minBackOffPeriod和maxBackOffPeriod，该策略在[minBackOffPeriod,maxBackOffPeriod之间取一个随机休眠时间，minBackOffPeriod默认500毫秒，maxBackOffPeriod默认1500毫秒；
 - ExponentialBackOffPolicy：指数退避策略，需设置参数sleeper、initialInterval、maxInterval和multiplier，initialInterval指定初始休眠时间，默认100毫秒，maxInterval指定最大休眠时间，默认30秒，multiplier指定乘数，即下一次休眠时间为当前休眠时间*multiplier；
 - ExponentialRandomBackOffPolicy：随机指数退避策略，引入随机乘数，之前说过固定乘数可能会引起很多服务同时重试导致DDos，使用随机休眠时间来避免这种情况。

```java
public interface BackOffPolicy {
    BackOffContext start(RetryContext context);
    void backOff(BackOffContext backOffContext) throws BackOffInterruptedException;
}
```
BackOffPolicy的接口定义如上，start方法会每调用一次 `excute(RetryCallback callbace)`时候执行一次，backOff会在两次重试的间隔间执行，即每次重试期间执行一次且最后一次重试后不再执行。具体可以参考源码：

```java
protected <T, E extends Throwable> T doExecute(RetryCallback<T, E> retryCallback,
        RecoveryCallback<T> recoveryCallback, RetryState state)
        throws E, ExhaustedRetryException {
    RetryPolicy retryPolicy = this.retryPolicy;       //重试策略
    BackOffPolicy backOffPolicy = this.backOffPolicy; // 退避策略        
    //重试上下文，主要记录了重试次数、上一次产生的异常、是否耗尽（例如重试是否超过最大次数）
    RetryContext context = open(retryPolicy, state);
    // 将上下文放入ThreadLocal
    RetrySynchronizationManager.register(context);

    Throwable lastException = null; // 上一次产生的异常
    boolean exhausted = false;      // 是否耗尽
    try {
        //拦截器模式，执行RetryListener#open方法
        boolean running = doOpenInterceptors(retryCallback, context);
        if (!running) {
            throw new TerminatedRetryException(
                    "Retry terminated abnormally by interceptor before first attempt");
        }
        // Get or Start the backoff context...
        BackOffContext backOffContext = null;
        Object resource = context.getAttribute("backOffContext");
        if (resource instanceof BackOffContext) {
            backOffContext = (BackOffContext) resource;
        }
        if (backOffContext == null) {
            // 调用backOffPolicy的start方法，并放入上下文，所以上面几行可以获取到。
            backOffContext = backOffPolicy.start(context);
            if (backOffContext != null) {
                context.setAttribute("backOffContext", backOffContext);
            }
        }

        /*
         * We allow the whole loop to be skipped if the policy or context already
         * forbid the first try. This is used in the case of external retry to allow a
         * recovery in handleRetryExhausted without the callback processing (which
         * would throw an exception).
         */
         //判断是否可以重试执行
        while (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
            try {
                // Reset the last exception, so if we are successful
                // the close interceptors will not think we failed...
                // 清除上一次的异常，执行RetryCallback回调
                lastException = null;
                return retryCallback.doWithRetry(context);
            }
            catch (Throwable e) {
                // 获得上一次的异常
                lastException = e;

                try {
                    // 遇到异常后，注册该异常的失败次数
                    registerThrowable(retryPolicy, state, context, e);
                }
                catch (Exception ex) {
                    throw new TerminatedRetryException("Could not register throwable",
                            ex);
                }
                finally {
                    // 执行RetryListener#onError
                    doOnErrorInterceptors(retryCallback, context, e);
                }
                // 如果可以重试，执行退避算法，比如休眠一小段时间后再重试
                if (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
                    try {
                        backOffPolicy.backOff(backOffContext);
                    }
                    catch (BackOffInterruptedException ex) {
                        lastException = e;
                        throw ex;
                    }
                }
                // 在有状态重试时，如果是需要执行回滚操作的异常，则立即抛出异常
                if (shouldRethrow(retryPolicy, context, state)) {
                    throw RetryTemplate.<E>wrapIfNecessary(e);
                }
            }
            /*
             * A stateful attempt that can retry may rethrow the exception before now,
             * but if we get this far in a stateful retry there's a reason for it,
             * like a circuit breaker or a rollback classifier.
             */
             // 如果是有状态重试，且有GLOBAL_STATE属性，则立即跳出重试终止；当抛出的异常是非需要执行回滚操作的异常时，才会执行到此处，CircuitBreakerRetryPolicy会在此跳出循环；
            if (state != null && context.hasAttribute(GLOBAL_STATE)) {
                break;
            }
        }
        // 清理环境
        exhausted = true;
        return handleRetryExhausted(recoveryCallback, context, state);
    }
    catch (Throwable e) {
        throw RetryTemplate.<E>wrapIfNecessary(e);
    }
    finally {
        // 清理环境
        close(retryPolicy, context, state, lastException == null || exhausted);
        // 执行RetryListener#close，比如统计重试信息
        doCloseInterceptors(retryCallback, context, lastException);
        RetrySynchronizationManager.clear();
    }
}
```
----------
# **有状态or无状态**

**无状态重试**，是在一个循环中执行完重试策略，即重试上下文保持在一个线程上下文中，在一次调用中进行完整的重试策略判断。
非常简单的情况，如远程调用某个查询方法时是最常见的无状态重试。
```java
RetryTemplate template = new RetryTemplate();
//重试策略：次数重试策略
RetryPolicy retryPolicy = new SimpleRetryPolicy(3);
template.setRetryPolicy(retryPolicy);
//退避策略：指数退避策略
ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
backOffPolicy.setInitialInterval(100);
backOffPolicy.setMaxInterval(3000);
backOffPolicy.setMultiplier(2);
backOffPolicy.setSleeper(new ThreadWaitSleeper());
template.setBackOffPolicy(backOffPolicy);

//当重试失败后，抛出异常
String result = template.execute(new RetryCallback<String, RuntimeException>() {
    @Override
    public String doWithRetry(RetryContext context) throws RuntimeException {
        throw new RuntimeException("timeout");
    }
});
//当重试失败后，执行RecoveryCallback
String result = template.execute(new RetryCallback<String, RuntimeException>() {
    @Override
    public String doWithRetry(RetryContext context) throws RuntimeException {
        System.out.println("retry count:" + context.getRetryCount());
        throw new RuntimeException("timeout");
    }
}, new RecoveryCallback<String>() {
    @Override
    public String recover(RetryContext context) throws Exception {
        return "default";
    }
});
```
**有状态重试**，有两种情况需要使用有状态重试，事务操作需要回滚或者熔断器模式。
事务操作需要回滚场景时，当整个操作中抛出的是数据库异常DataAccessException，则不能进行重试需要回滚，而抛出其他异常则可以进行重试，可以通过RetryState实现：
```java
//当前状态的名称，当把状态放入缓存时，通过该key查询获取
Object key = "mykey";
//是否每次都重新生成上下文还是从缓存中查询，即全局模式（如熔断器策略时从缓存中查询）
boolean isForceRefresh = true;
//对DataAccessException进行回滚
BinaryExceptionClassifier rollbackClassifier =
        new BinaryExceptionClassifier(Collections.<Class<? extends Throwable>>singleton(DataAccessException.class));
RetryState state = new DefaultRetryState(key, isForceRefresh, rollbackClassifier);

String result = template.execute(new RetryCallback<String, RuntimeException>() {
    @Override
    public String doWithRetry(RetryContext context) throws RuntimeException {
        System.out.println("retry count:" + context.getRetryCount());
        throw new TypeMismatchDataAccessException("");
    }
}, new RecoveryCallback<String>() {
    @Override
    public String recover(RetryContext context) throws Exception {
        return "default";
    }
}, state);
```
RetryTemplate中在有状态重试时，回滚场景时直接抛出异常处理代码：
```java
//state != null && state.rollbackFor(context.getLastThrowable())
//在有状态重试时，如果是需要执行回滚操作的异常，则立即抛出异常
if (shouldRethrow(retryPolicy,context, state)) {
    throw RetryTemplate.<E>wrapIfNecessary(e);
}
```
熔断器场景。在有状态重试时，且是全局模式，不在当前循环中处理重试，而是全局重试模式（不是线程上下文），如熔断器策略时测试代码如下所示。
```java
RetryTemplate template = new RetryTemplate();
CircuitBreakerRetryPolicy retryPolicy =
        new CircuitBreakerRetryPolicy(new SimpleRetryPolicy(3));
retryPolicy.setOpenTimeout(5000);
retryPolicy.setResetTimeout(20000);
template.setRetryPolicy(retryPolicy);

for (int i = 0; i < 10; i++) {
    try {
        Object key = "circuit";
        boolean isForceRefresh = false;
        RetryState state = new DefaultRetryState(key, isForceRefresh);
        String result = template.execute(new RetryCallback<String, RuntimeException>() {
            @Override
            public String doWithRetry(RetryContext context) throws RuntimeException {
                System.out.println("retry count:" + context.getRetryCount());
                throw new RuntimeException("timeout");
            }
        }, new RecoveryCallback<String>() {
            @Override
            public String recover(RetryContext context) throws Exception {
                return "default";
            }
        }, state);
        System.out.println(result);
    } catch (Exception e) {
        System.out.println(e);
    }
}
```
为什么说是全局模式呢？我们配置了isForceRefresh为false，则在获取上下文时是根据key “circuit”从缓存中获取，从而拿到同一个上下文。
```java
Object key = "circuit";
boolean isForceRefresh = false;
RetryState state = new DefaultRetryState(key,isForceRefresh);

如下RetryTemplate代码说明在有状态模式下，不会在循环中进行重试。
if (state != null && context.hasAttribute(GLOBAL_STATE)) {
   break;
}
```
熔断器策略配置代码，CircuitBreakerRetryPolicy需要配置三个参数：

> delegate：是真正判断是否重试的策略，当重试失败时，则执行熔断策略；
> openTimeout：openWindow，配置熔断器电路打开的超时时间，当超过openTimeout之后熔断器电路变成半打开状态（主要有一次重试成功，则闭合电路）；
> resetTimeout：timeout，配置重置熔断器重新闭合的超时时间。

判断熔断器电路是否打开的代码：
```java
public boolean isOpen() {
   long time = System.currentTimeMillis() - this.start;
   boolean retryable = this.policy.canRetry(this.context);
   if (!retryable) {//重试失败
      //在重置熔断器超时后，熔断器器电路闭合，重置上下文
      if (time > this.timeout) {
         this.context = createDelegateContext(policy, getParent());
         this.start = System.currentTimeMillis();
         retryable = this.policy.canRetry(this.context);
      } else if (time < this.openWindow) {
         //当在熔断器打开状态时，熔断器电路打开，立即熔断
         if ((Boolean) getAttribute(CIRCUIT_OPEN) == false) {
            setAttribute(CIRCUIT_OPEN, true);
         }
         this.start = System.currentTimeMillis();
         return true;
      }
   } else {//重试成功
      //在熔断器电路半打开状态时，断路器电路闭合，重置上下文
      if (time > this.openWindow) {
         this.start = System.currentTimeMillis();
         this.context = createDelegateContext(policy, getParent());
      }
   }
   setAttribute(CIRCUIT_OPEN, !retryable);
   return !retryable;
}
```
从如上代码可看出spring-retry的熔断策略相对简单：

当重试失败，且在熔断器打开时间窗口[0,openWindow) 内，立即熔断；
当重试失败，且在指定超时时间后(>timeout)，熔断器电路重新闭合；
在熔断器半打开状态[openWindow, timeout] 时，只要重试成功则重置上下文，断路器闭合。
CircuitBreakerRetryPolicy的delegate应该配置基于次数的SimpleRetryPolicy或者基于超时的TimeoutRetryPolicy策略，且策略都是全局模式，而非局部模式，所以要注意次数或超时的配置合理性。

    [请求可以通过]closed 
            -> [fail:1 fail:2 fail:3] ->    
    [请求不能通过]open 
            -> [在openTimeout有成功请求] ->  
    [请求部分通过]half-open
            -> [在resetTimeout没有失败请求] ->  
    [请求可以通过]closed 


比如SimpleRetryPolicy配置为3次，openWindow=5s，timeout=20s，我们来看下CircuitBreakerRetryPolicy的极端情况。

特殊时间序列：

1s：retryable=false，重试失败，断路器电路处于打开状态，熔断，重置start时间为当前时间；
2s：retryable=false，重试失败，断路器电路处于打开状态，熔断，重置start时间为当前时间；
7s：retryable=true，表示可以重试，但是time=5s，time > this.openWindow判断为false，CIRCUIT_OPEN=false，不熔断；此时重试次数=3，等于最大重试次数了；
10s：retryable=false，因重试次数>3，time=8s，time < this.openWindow判断为false，熔断，且在timeout超时之前都处于熔断状态，这个时间段要配置好，否则熔断的时间会太长（默认timeout=20s）；
(7s,20s]之间的所有重试：和10s的情况一样。
如上是当重试次数正好等于最大重试次数，且time=openWindow时的特殊情况，不过实际场景这种情况几乎不可能发生。

spring-retry的重试机制没有像Hystrix根据失败率阀值进行电路打开/关闭的判断。

如果需要局部循环重试机制，需要组合多个RetryTemplate实现。

spring-retry也提供了注解实现：
@EnableRetry、@Retryable、@Recover、@Backoff、@CircuitBreaker。具体可以参考官方文档。

#**统计分析**
spring-retry通过RetryListener实现拦截器模式，默认提供了StatisticsListener实现重试操作统计分析数据。
```java
RetryTemplatetemplate = new RetryTemplate();
DefaultStatisticsRepository repository = new DefaultStatisticsRepository();
StatisticsListener listener = new StatisticsListener(repository);
template.setListeners(new RetryListener[]{listener});

for (int i = 0; i < 10; i++){
    String result = template.execute(new RetryCallback<String, RuntimeException>() {
        @Override
       public String doWithRetry(RetryContext context) throws RuntimeException {
           context.setAttribute(RetryContext.NAME,"method.key");
            return "ok";
        }
    });
}
RetryStatistics statistics = repository.findOne("method.key");
System.out.println(statistics);
```
此处要给操作定义一个name如“method.key”，从而查询该操作的统计分析数据。

到此spring-retry重试与熔断就介绍完了。spring-retry项目地址https://github.com/spring-projects/spring-retry。

另外可以参考《亿级流量网站架构核心技术》的《第5章 降级特技》和《第6章 超时与重试机制》了解和学习更多内容。

## **参考**
- [spring-retry重试与熔断详解—《亿级流量》内容补充](http://www.broadview.com.cn/article/233)
- [使用熔断器设计模式保护软件](https://www.cnblogs.com/shanyou/p/CircuitBreaker.html)
- [CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)