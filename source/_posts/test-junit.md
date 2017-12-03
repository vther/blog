---
title: 单元测试 之 JUnit
date: 2017年12月3日16:19:59
thumbnail: http://junit.org/junit4/images/junit-logo.png
tags: 
 - 单元测试
 - JUnit
categories: 
 - 单元测试
---

单元测试是在编程和重构中被极力推荐使用的，可以大大的提高开发的效率（这里的效率需要综合计算后续可维护性，已经出现BUG重复修改等下，而不简单单是开发时间）。但是实际上编写测试代码也是需要耗费很多的时间和精力的，有的时候甚至要比编写代码本身花费的时间还要多，所以如何写好单元测试用例是一门很深奥的学问。

好的测试用例应该具有以下四种特性：

 1. 正确性
程序中的每一项功能都是测试来验证它的正确性，是后续维护代码、重构代码的保证。
 2. 设计性
编写单元测试将使我们从调用者观察、思考。特别是先写测试（test-first），迫使我们把程序设计成易于调用和可测试的，即迫使我们解除软件中的耦合。
 3. 指导性
单元测试应该具有指导性，类似于文档，是对函数或类使用的最佳实践。而且，这份文档是可编译、可运行的，并且它保持最新，永远与代码同步。
 4. 回归性
自动化的单元测试避免了代码出现回归，编写完成之后，可以随时随地的快速运行测试。

后续我将用一系列的博客，来记录如何对于Java代码来做单元测试，本文先从最基础的JUnit做起，先来学习基本的工具的使用。
<!--more-->
### JUnit 功能点
#### 1，断言 - [Assertions](https://github.com/junit-team/junit4/wiki/Assertions)
JUnit支持对数组、对象的断言,可以通过`assertArrayEquals`、`assertEquals`、`assertFalse`、`assertNotNull`、`assertNotSame`、`assertNull`、`assertSame` 等方法进行比较，
JUnit还支持更复杂的Matchers用法，是通过集成了[hamcrest](http://mvnrepository.com/artifact/junit/junit/4.12)的jar包来实现了如下功能，源码如下，可见，采用直接调用调用的方式：
```java
public static <T> void assertThat(String reason, T actual, Matcher<? super T> matcher) {
    MatcherAssert.assertThat(reason, actual, matcher);
}
```
示例
```java
import static org.hamcrest.CoreMatchers.*;
import static org.junit.Assert.*;
import org.hamcrest.core.CombinableMatcher;
import org.junit.Test;

public class AssertionsTest {

    // Simple use
    @Test
    public void testAssert() {
        byte[] expected = "trial".getBytes();
        byte[] actual = "trial".getBytes();
        assertArrayEquals("failure - byte arrays not same", expected, actual);
        
        assertNotSame("should not be same Object", new Object(), new Object());
    }
 
    // Core Hamcrest Matchers with assertThat
    @Test
    public void testAssertThatBothContainsString() {
        assertThat("albumen", both(containsString("a")).and(containsString("b")));
        assertThat(Arrays.asList("one", "two", "three"), hasItems("one", "three"));
        assertThat(Arrays.asList("fun", "ban", "net"), everyItem(containsString("n")));
        assertThat("good", allOf(equalTo("good"), startsWith("good")));
        assertThat("good", not(allOf(equalTo("bad"), equalTo("good"))));
        assertThat("good", anyOf(equalTo("bad"), equalTo("good")));
        assertThat(7, not(CombinableMatcher.<Integer>either(equalTo(3)).or(equalTo(4))));
        assertThat(new Object(), not(sameInstance(new Object())));
    }
}
```
**TIPS:**
1. 关于[Matchers and assertThat](https://github.com/junit-team/junit4/wiki/Matchers-and-assertthat)，这里提供了更细致的分析。
2. 有些第三方的扩展Mather也很有意思，例如[JSON Matcher](https://github.com/hertzsprung/hamcrest-json)
```java
assertThat("{\"age\":43, \"friend_ids\":[16, 52, 23]}",
            sameJSONAs("{\"friend_ids\":[52, 23, 16]}")
                .allowingExtraUnexpectedFields()// 允许不识别的额外字段
                .allowingAnyArrayOrdering());// 允许任意顺序
```
#### 2，测试执行器 - [Test runners](https://github.com/junit-team/junit4/wiki/Test-runners)

当一个测试类使用@RunWith注解时，JUnit会调用注解中的类执行测试用例。JUnit默认使用的Runner是`JUnit4.class`，还提供了`Suite.class`、`Parameterized.class`、`Categories.class`，我将在本文依次介绍。当然我们还有其他常用的来自于第三方提供的如`SpringJUnit4ClassRunner.class`、`MockitoJUnitRunner.class`、`PowerMockRunner.class`，我们后续也会说到。

#### 3，打包执行器 - [Aggregating tests in Suites](https://github.com/junit-team/junit4/wiki/Aggregating-tests-in-suites)
使用`@RunWith(Suite.class)`可以将多个用例打包，放在一起执行。示例如下：
```java
import org.junit.runner.RunWith;
import org.junit.runners.Suite;

@RunWith(Suite.class)
@Suite.SuiteClasses({
  TestFeatureLogin.class,
  TestFeatureLogout.class,
  TestFeatureNavigate.class,
  TestFeatureUpdate.class
})
public class FeatureTestSuite {
  // 这个类留空即可，仅仅是最为上述注解的容器使用
}
```
#### 4，参数化执行器 - [Parameterized tests](https://github.com/junit-team/junit4/wiki/Parameterized-tests)
使用`@RunWith(Parameterized.class)`可以像函数一样，传入多个入参和出参的对应，并逐一校验是否正确。例如，下面的取和函数`int sum(int a, int b)`，我们可能想测试1+2=3,2+3=5,3+6=9等多种，如果要一个一个写@Test，比较繁琐，这时候可以使用参数化来简化代码书写。
```
@RunWith(Parameterized.class)
public class ParameterizedTest {
    @Parameters
    public static Collection<Object[]> data() {
        return Arrays.asList(new Object[][]{
                {1, 2, 3}, {2, 3, 5}, {3, 6, 9}
        });
    }

    private int a;
    private int b;
    private int result;
    // 构造函数和@Parameters注解标注的集合的赋值顺序必须一致
    public ParameterizedTest(int a, int b, int result) {
        this.a = a;
        this.b = b;
        this.result = result;
    }
    
    @Test
    public void test() {
        assertEquals(result, sum(a, b));
    }

    private int sum(int a, int b) {
        return a + b;
    }
}
```
**TIPS：**
1，如果不想使用构造函数，还可以使用`@Parameter`注解来替代构造函数，但**需要被注解的参数是public的**
```java
@Parameter // 相当于@Parameter(0)
public int fInput;
@Parameter(1)
public int fExpected;
```
2，单个参数的时候，`@Parameters`标注的方法可以稍作简化(版本要大于4.12-beta-3)。
```
@Parameters
public static Iterable<? extends Object> data() {
    return Arrays.asList("first test", "second test");
}
```
#### 5，分类执行器 - [Categories](https://github.com/junit-team/junit4/wiki/Categories)
使用`@RunWith(Categories.class)`可以分类，使用方法类似Hibernate Validator的Group，fastxml的MixInAnnotations.
```java
public interface FastTests { /* 定义空接口，用于分类标记 */}
public interface SlowTests { /* 定义空接口，用于分类标记 */}

public class A {
    @Test
    public void a() {
        fail();
    }
    @Category(SlowTests.class)
    @Test
    public void b() {}
}

@Category({SlowTests.class, FastTests.class})
public class B {
    @Test
    public void c() {}
}

@RunWith(Categories.class)
@Categories.IncludeCategory(SlowTests.class)
@Suite.SuiteClasses({A.class, B.class}) // Categories也是一种Suite
public class SlowTestSuite {
    // A.b 和 B.c 会执行，A.a不会
}

@RunWith(Categories.class)
@Categories.IncludeCategory(SlowTests.class)
@Categories.ExcludeCategory(FastTests.class)
@Suite.SuiteClasses({A.class, B.class}) // Categories也是一种Suite
public class SlowTestSuite {
    // A.b会执行, A.a 和 B.c 不会
}
```

#### 6，用例执行顺序 - [Test execution order](https://github.com/junit-team/junit4/wiki/Test-execution-order)
自动JUnit4.11，使用`@FixMethodOrder`可以标改变用例的执行顺序，但是一般不会使用，因为如果用例有执行依赖，那么用例必定设定的不太合适。JUnit目前提供了以下几种默认实现：
`MethodSorters.DEFAULT` ： 默认是按照用例函数名的hashcode比较
`MethodSorters.JVM` : 让JVM决定顺序
`MethodSorters.NAME_ASCENDING` ： 按照用例函数名称升序

#### 7，异常测试 - [Exception testing](https://github.com/junit-team/junit4/wiki/Exception-testing)
使用`@RunWith(Categories.class)`可以分类，使用方法类似Hibernate Validator的Group，fastxml的MixInAnnotations.
```java
@Test(expected = IndexOutOfBoundsException.class) 
public void empty() { 
     new ArrayList<Object>().get(0); 
}
```
或者使用try catch
```java
@Test
public void testExceptionMessage() {
    try {
        new ArrayList<Object>().get(0);
        fail("Expected an IndexOutOfBoundsException to be thrown");
    } catch (IndexOutOfBoundsException anIndexOutOfBoundsException) {
        assertThat(anIndexOutOfBoundsException.getMessage(), is("Index: 0, Size: 0"));
    }
}
```
或者使用`ExpectedException Rule`，Rule的作用会在下面说，暂时可以理解为，一种加在所有测试方法上的校验，下面的每一个用例都得满足这个条件。
```java
@Rule public ExpectedException thrown = ExpectedException.none();

@Test
public void shouldTestExceptionMessage() throws IndexOutOfBoundsException {
    List<Object> list = new ArrayList<Object>();
 
    thrown.expect(IndexOutOfBoundsException.class);
    thrown.expectMessage("Index: 0, Size: 0");
    // 或者可以使用Matchers来匹配
    // thrown.expectMessage(Matchers.containsString("Size: 0"));
    list.get(0); // execution will never get past this line
}
```

#### 8，超时测试 - [Timeout for tests](https://github.com/junit-team/junit4/wiki/Timeout-for-tests)
很简单，直接看代码喽
```java
@Test(timeout=1000)
public void testWithTimeout() {}
```
使用`Timeout Rule`，校验这个类的所有方法的运行时间都不能超过10秒钟，不然就会标志为失败。
这里需要注意的一点是Rule需要是public field
```java
public class HasGlobalTimeout {
    @Rule public Timeout globalTimeout = Timeout.seconds(10); // 每个方法10s超时

    @Test
    public void testSleepForTooLong() throws Exception {
        TimeUnit.SECONDS.sleep(100); // 休眠100s，会执行失败
    }

    @Test
    public void testBlockForever() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);
        latch.await(); // 永久阻塞 ，会执行失败
    }
}
```
**TIPS：**
1，需要注意的是，Timeout Rule的时间是包括@Before和@After的时间的，所以，**@After方法是存在不被执行的可能的。如果想在@After里面清空或者释放资源，要注意了。**

#### 9，规则 - [Rules](https://github.com/junit-team/junit4/wiki/Rules)
一个JUnit Rule就是一个实现了TestRule的类，这些类的作用有点类似于@Before、@After（是用来在每个测试方法的执行前后执行一些代码的一个方法），但是Rule更倾向于校验。是指满足某种条件用例才会执行成功的一种校验。例如上面提到的`ExpectedException Rule`,`Timeout Rule`，分别致力于异常校验、超时校验，JUnit还提供了其他的Rule，如：

 - `TemporaryFolder`：规则允许创建文件或文件夹，在测试方法(无论是通过还是失败)执行完成时会被删除。默认情况下，如果资源无法被删除是不会抛出异常的。[参考](http://www.importnew.com/15887.html)
 - `ExternalResource`：ExternalResource是规则(例如TemporaryFolder)的基础类，在测试之前建立一个外部资源(一个文件，套接字[socket]，服务[server]，数据库连接，等等),并且保证tear it down.
 - `ErrorCollector`：ErrorCollector 规则允许在第一个问题发现之后继续运行(例如，收集下面不正确的行到一个表中，然后一次性报告所有错误。)
 - `Verifier`：Verifier是规则(例如ErrorCollector)的一个基类，无论测试的执行情况是什么，只要验证框架不通过，则视为测试失败。
 - `TestWatchman/TestWatcher`：TestWatchman 是JUnit 4.7引入的，但是已经过时，被 TestWatcher [4.9引入] 所取代。TestWatcher 是一个规则的基础类,that take note of the testing action，而不用修改测试类。例如，这个类将保存一个日志在一个通过以及失败的测试：
 - `TestName`：TestName 规则使得当前测试的名字在测试内部是可利用的：
 - `ClassRule `：The ClassRule annotation extends the idea of method-level Rules, adding static fields that can affect the operation of a whole class. Any subclass of ParentRunner, including the standard BlockJUnit4ClassRunner and Suite classes, will support ClassRules.
 - `RuleChain`：规则链，多个规则的组合
 - `Custom Rules`：自定义规则
 

#### 10，忽略测试 - [Ignoring tests](https://github.com/junit-team/junit4/wiki/Ignoring-tests)
使用@Ignore注解即可

#### 11，假设 - [Assumptions with assume](https://github.com/junit-team/junit4/wiki/Assumptions-with-assume)
假设有点类似于java里面的if，如果`assumeThat`不成立，**后面的代码将不会执行，且UT不会报错。**
```java
import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assume.assumeThat;

public class AssumptionsTest {
    @Test
    public void assume() {
        String filePath = "parent" + File.separator + "child.cfg";
        assumeThat(File.separatorChar, is("/"));
        // 只有assumeThat成立的时候，才能执行这里的断言，assumeThat失败了，则用例不会失败
        assertThat(filePath, is("parent/child.cfg"));
    }
}
```


#### 12，测试夹具 - [Test fixtures](https://github.com/junit-team/junit4/wiki/Test-fixtures)
Test fixtures，一般用于在测试前后：
1，准备输入数据、创建Mock对象
2，从数据库中取数据
3，做一些公用的操作

其执行流程一般为：
```java
@BeforeClass

@Before
@Test   // test2()方法
@After

@Before 
@Test   // test1()方法
@After 

@AfterClass 
```


#### 13，Theories - [Theories](https://github.com/junit-team/junit4/wiki/Theories)

Theories是指更灵活且富有表现力的断言(assertions)和假设(assumptions)。@Test集中于一种特定的场景，而@Theory则可能覆盖无限的潜在场景。很多情况下，Theories会比`Parameterized.class`更好用，因为写法更简单。

例如如下测试，每一个`@DataPoint`标注的变量都会去执行`@Theory`标注的测试，所以，该方法执行了两次。带有`/`的名字是不测试无法通过`assumeThat`,其测试结果被忽略
```java

import org.junit.experimental.theories.DataPoint;
import org.junit.experimental.theories.Theories;
import org.junit.experimental.theories.Theory;
import org.junit.runner.RunWith;

import static org.hamcrest.CoreMatchers.containsString;
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.core.IsNot.not;
import static org.junit.Assume.assumeThat;

@RunWith(Theories.class)
public class TheoriesTest {
    @DataPoint
    public static String GOOD_USERNAME = "optimus";
    @DataPoint
    public static String USERNAME_WITH_SLASH = "optimus/prime";

    @Theory
    public void filenameIncludesUsername(String username) {
        assumeThat(username, not(containsString("/")));
        assertThat(new User(username).configFileName(), containsString(username));
    }

    private class User {
        String username;
        String configFileName;

        User(String username) {
            this.username = username;
            this.configFileName = "config/" + username;
        }
        String configFileName() {
            return configFileName;
        }
    }
}
```
多参数测试
```java
package com.vther.junit;

import org.junit.Test;
import org.junit.experimental.theories.Theories;
import org.junit.experimental.theories.Theory;
import org.junit.experimental.theories.suppliers.TestedOn;
import org.junit.runner.RunWith;

import static org.hamcrest.CoreMatchers.not;
import static org.hamcrest.core.Is.is;
import static org.junit.Assert.assertThat;
import static org.junit.Assume.assumeThat;

@RunWith(Theories.class)
public class DollarTest {
    // 普通测试
    @Test
    public void multiplyByAnInteger() {
        assertThat(new Dollar(5).times(2).getAmount(), is(10));
    }
    // Theory测试，会测试3*3=9次，m=0的会略过
    @Theory
    public void multiplyIsInverseOfDivideWithInlineDataPoints(
            @TestedOn(ints = {0, 5, 10}) int amount,
            @TestedOn(ints = {0, 1, 2}) int m) {
        assumeThat(m, not(0));
        assertThat(new Dollar(amount).times(m).divideBy(m).getAmount(), is(amount));
    }

    class Dollar {
        private int amount;
        Dollar(int amount) {
            this.amount = amount;
        }
        Dollar times(int i) {
            amount = amount * i;
            return this;
        }
        Dollar divideBy(int m) {
            this.amount = amount / m;
            return this;
        }        
        int getAmount() {
            return amount;
        }
    }
}

```
当然我们可以使用自定义注解来测试一个区间，比如金额在-100到100的金额测试，我们需要按照以下步骤来做：
```
    // 1，自定义一个注解，使用@ParametersSuppliedBy定义其参数来源
    @Retention(RetentionPolicy.RUNTIME)
    @ParametersSuppliedBy(BetweenSupplier.class)
    public @interface Between {
        int first(); // 起始值
        int last(); // 结束值
    }
    // 2，定义参数来源
    public static class BetweenSupplier extends ParameterSupplier {
        @Override
        public List<PotentialAssignment> getValueSources(ParameterSignature sig) {
            Between annotation = sig.getAnnotation(Between.class);
            return IntStream.rangeClosed(annotation.first(), annotation.last())
                    .mapToObj(i -> PotentialAssignment.forValue("ints", i))
                    .collect(Collectors.toList());
    
        }
    }
    // 3，进行测试
    @Theory
    public void multiplyIsInverseOfDivideWithInlineDataPointsByCustomAnnotation(
            @Between(first = -100, last = 100) int amount,
            @Between(first = -100, last = 100) int m) {
        assumeThat(m, not(0));
        assertThat(new Dollar(amount).times(m).divideBy(m).getAmount(), is(amount));
    }
```

#### 14，多线程并发测试 - [Multithreaded code and concurrency](https://github.com/junit-team/junit4/wiki/Multithreaded-code-and-concurrency)
JUnit对多线程的支持很差，我们可以使用[ConcurrentUnit](https://github.com/jhalterman/concurrentunit)来测试。但是如果只是简单测试多个线程能不能在固定时间内完成，可以使用以下方法：
```
public static void assertConcurrent(final String message, final List<? extends Runnable> runnables, final int maxTimeoutSeconds) throws InterruptedException {
    final int numThreads = runnables.size();
    final List<Throwable> exceptions = Collections.synchronizedList(new ArrayList<Throwable>());
    final ExecutorService threadPool = Executors.newFixedThreadPool(numThreads);
    try {
        final CountDownLatch allExecutorThreadsReady = new CountDownLatch(numThreads);
        final CountDownLatch afterInitBlocker = new CountDownLatch(1);
        final CountDownLatch allDone = new CountDownLatch(numThreads);
        for (final Runnable submittedTestRunnable : runnables) {
            threadPool.submit(new Runnable() {
                public void run() {
                    allExecutorThreadsReady.countDown();
                    try {
                        afterInitBlocker.await();
                        submittedTestRunnable.run();
                    } catch (final Throwable e) {
                        exceptions.add(e);
                    } finally {
                        allDone.countDown();
                    }
                }
            });
        }
        // wait until all threads are ready
        assertTrue("Timeout initializing threads! Perform long lasting initializations before passing runnables to assertConcurrent", allExecutorThreadsReady.await(runnables.size() * 10, TimeUnit.MILLISECONDS));
        // start all test runners
        afterInitBlocker.countDown();
        assertTrue(message +" timeout! More than" + maxTimeoutSeconds + "seconds", allDone.await(maxTimeoutSeconds, TimeUnit.SECONDS));
    } finally {
        threadPool.shutdownNow();
    }
    assertTrue(message + "failed with exception(s)" + exceptions, exceptions.isEmpty());
}
```
#### 14，测试用例统计 - [maven-surefire-plugin](http://maven.apache.org/surefire/maven-surefire-plugin/)
使用maven-surefire-plugin可以完成对JUnit的用例覆盖率统计等功能。具体可以参考maven插件说明。

#### 15，持续集成 - [Continuous testing](https://github.com/junit-team/junit4/wiki/Continuous-testing)

使用[Infinitest插件](http://infinitest.github.io/)即可完成每次修改代码都进行UT测试。

