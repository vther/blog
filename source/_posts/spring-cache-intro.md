---
title: Spring Cache 入门
date: 2017年11月29日21:19:59
thumbnail: https://spring.io/img/homepage/icon-spring-framework.svg
tags: 
 - Spring
 - Spring Cache
categories: 
 - Spring
---

缓存在很多场景下都是相当有用的。例如，计算或检索一个值的代价很高，并且对同样的输入需要不止一次获取值的时候，就应当考虑使用缓存。

通常来说，缓存适用于：

 1. 你愿意消耗一些内存空间来提升速度。
 2. 你预料到某些键会被查询一次以上。
 3. 缓存中存放的数据总量不会超出内存容量。
 4. 如果你的场景符合上述的每一条，就适合使用缓存。

Spring Cache 可以基于注释（annotation）配置，对业务代码侵入极小，但却具备相当的灵活性，不仅能够使用 SpEL（Spring Expression Language）来定义缓存的 key 和各种 condition，还提供开箱即用的缓存临时存储方案，也支持和主流的专业缓存例如 EHCache 集成。

<!--more-->
## 缓存名词解释

#### 缓存命中率（Hit rate）
即从缓存中读取数据的次数 与 总读取次数的比率，命中率越高越好：
> 命中率 = 从缓存中读取次数 / (总读取次数[从缓存中读取次数 + 从慢速设备上读取的次数]) 
> Miss率 = 没有从缓存中读取的次数 /(总读取次数[从缓存中读取次数 + 从慢速设备上读取的次数])

这是一个非常重要的监控指标，如果做缓存一定要健康这个指标来看缓存是否工作良好；
 
#### 缓存策略（Eviction policy）
移除策略，即如果缓存满了，从缓存中移除数据的策略；常见的有LFU、LRU、FIFO：
FIFO（First In First Out）：先进先出算法，即先放入缓存的先被移除；
LRU（Least Recently Used）：最久未使用算法，使用时间距离现在最久的那个被移除；
LFU（Least Frequently Used）：最近最少使用算法，一定时间段内使用次数（频率）最少的那个被移除；
 
#### TTL（Time To Live ）
存活期，即从缓存中创建时间点开始直到它到期的一个时间段（不管在这个时间段内有没有访问都将过期）
 
#### TTI（Time To Idle）
空闲期，即一个数据多久没被访问将从缓存中移除的时间。

## Spring Cache 配置和使用

最简单的配置只需要两步：
```java
// 启用缓存
@Configuration
@EnableCaching
public class CacheConfig {
}

// 使用缓存
@Component
public class CacheService {

    // 查询后，只要缓存不过期，即可略过函数方法，直接返回结果
    @Cacheable(cacheNames = {"books"}, key = "#id")
    public Book findBook(int id) {
        // Do find 
        return book;
    }

    // 不管有无缓存，方法每次都执行，适用于更新的场景（可以避免去重新查询一次）
    @CachePut(cacheNames = {"books", "books2"}, key = "#book.id")
    public Book updateBook(Book book) {
        // Do update
        return book;
    }

    // 清除缓存
    @CacheEvict(cacheNames = {"books", "books2"}, key = "#book.id")
    public Book deleteBook(Book book) {
        // Do delete
        return book;
    }
}
```
## Spring Cache 注解详解

| 注解        | 释义           | 
| ------------- |:-------------| 
| @EnableCaching| 启用缓存 |  
| @CacheConfig| 主要用于配置该类中会用到的一些共用的缓存配置。在这里@CacheConfig(cacheNames = "users")：配置了该数据访问对象中返回的内容将存储于名为users的缓存对象中，我们也可以不使用该注解，直接通过@Cacheable自己配置缓存集的名字来定义。| 
| @Cacheable | 配置了函数的返回值将被加入缓存。同时在查询时，会先从缓存中获取，若不存在才再发起对数据库的访问。  |       
| @CachePut | 配置于函数上，能够根据参数定义条件来进行缓存，它与@Cacheable不同的是，它每次都会真是调用函数，所以主要用于数据新增和修改操作上。它的参数与@Cacheable类似，具体功能可参考上面对@Cacheable参数的解析 |        
| @CacheEvict | 配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。 |  
| @Caching | @Cacheable @CachePut @CacheEvict的复合（combine）操作 |  

拥有属性详解：

| 注解| 拥有属性| 释义  |
| ------------- |:-------------| ----- |
| @Cacheable @CachePut @CacheEvict | value|用于指定缓存存储的集合名。如果使用@CacheConfig指定缓存名，则可不填 |
| @CacheConfig @Cacheable @CachePut @CacheEvict | cacheNames  | cacheNames为value的别名 |
| @Cacheable @CachePut @CacheEvict | key | 缓存对象存储在Map集合中的key值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：@Cacheable(key > = "#p0")：使用函数第一个参数作为缓存的key值，更多关于SpEL表达式的详细内容可参考[官方文档](https://docs.spring.io/spring-framework/docs/4.3.9.RELEASE/spring-framework-reference/htmlsingle/#cache-spel-context) |
| @Cacheable @CachePut @CacheEvict| condition| 缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：@Cacheable(key= "#p0", condition = "#p0.length() < 3")，表示只有当第一个参数的长度小于3的时候才会被缓存，若做此配置上面的AAA用户就不会被缓存|
| @Cacheable @CachePut @CacheEvict | unless| 另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于condition参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对result进行判断。 |
| @CacheConfig @Cacheable @CachePut @CacheEvict | keyGenerator| 用于指定key生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现org.springframework.cache.interceptor.KeyGenerator接口，并使用该参数来指定。需要注意的是：该参数与key是互斥的 |
| @CacheConfig @Cacheable @CachePut @CacheEvict | cacheManager| 用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用 |
| @CacheConfig @Cacheable @CachePut @CacheEvict | cacheResolver| 用于指定使用那个缓存解析器，非必需。需通过org.springframework.cache.interceptor.CacheResolver接口来实现自己的缓存解析器，并用该参数指定。 |
| @Cacheable | sync| 多个线程并发加载同一个key，控制是否同步，默认false，设为ture有以下三点限制：不能使用unless、只能指定一个cache，不允许和其他缓存操作合并使用 |
| @CacheEvict  | beforeInvocation | 非必需，默认为false，会在调用方法之后移除数据。当为true时，会在调用方法之前移除数据。|
| @CacheEvict  | allEntries |非必需，默认为false。当为true时，会移除所有数据 |

## 擦除缓存

**手动清除**：我们可以自行调用如下方法
```java
@Component
public class CacheService {
    // 全部清除
    @CacheEvict(cacheNames = {"books"}, allEntries = true)
    public void evictAllCache() {}
    
    // 指定key清除
    @CacheEvict(cacheNames = {"books"}, key = "#id")
    public void evictCacheById() {int id}
}
```
**自动清除**：我们可以使用定时清除、或者为缓存设置过期策略等
例如，如下代码可以完成定时清除的功能。
```java
// 引入定时功能
@Configuration
@EnableScheduling
public class CacheConfig {}

// 自动清除
@Component
public class CacheService {
    @Scheduled(fixDelay = 1000)
    @CacheEvict(cacheNames = {"books"}, allEntries = true)
    public void scheduledEvict() {}

}
```
当然我们可以使用多个CacheManager，为每一个指定不同的定时执行频率来清除缓存。
```java
@Configuration
@EnableCaching
@EnableScheduling
public class CacheConfig {
    // 主缓存管理器
    @Primary
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("books", "books2");
    }

    @Bean
    public CacheManager cacheManager2() {
        return new ConcurrentMapCacheManager("books3");
    }
}

@Component
public class CacheService {

    @Scheduled(fixedDelay = 1000)
    @CacheEvict(cacheManager = "cacheManager", cacheNames = {"books", "books2"}, allEntries = true)
    public void scheduledEvict() {}

    @Scheduled(fixedDelay = 2000)
    @CacheEvict(cacheManager = "cacheManager2", cacheNames = {"books3"}, allEntries = true)
    public void scheduledEvict2() {}

}

```
虽然`@Scheduled`支持固定频率（fixRate）、固定延时（fixDelay）、cron表达式，但是带有`@Scheduled`注解的函数不能添加入参，所以在自动清除方面我们只能实现集合级别的清除，不能实现key级别的清除。其原因是Spring默认使用的的`CacheManager`是`ConcurrentMapCacheManager`，其功能有限。
**后续我将说明如何使用第三方缓存实现为Spring Cache（如Guava Cache，Redis等）实现该功能。**

## 参考

 1. [Spring Cache官网文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache)
 2. [Spring Cache抽象详解](http://jinnianshilongnian.iteye.com/blog/2001040)
 3. [注释驱动的 Spring cache 缓存介绍](https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-cache/)
 4. [一张图让你秒懂Spring @Scheduled定时任务的fixedRate,fixedDelay,cron执行差异](http://blog.csdn.net/applebomb/article/details/52400154)