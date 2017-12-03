---
title: 单元测试 之 Mockito
date: 2017年12月3日16:19:59
thumbnail: https://github.com/mockito/mockito.github.io/raw/master/img/logo%402x.png
tags: 
 - 单元测试
 - Mockito
categories: 
 - 单元测试
---

在软件开发中提及”mock”，通常理解为模拟对象。

为什么需要模拟? 在我们一开始学编程时,我们所写的对象通常都是独立的，并不依赖其他的类，也不会操作别的类。但实际上，软件中是充满依赖关系的，比如我们会基于service类写操作类,而service类又是基于数据访问类(DAO)的，依次下去，形成复杂的依赖关系。

单元测试的思路就是我们想在不涉及依赖关系的情况下测试代码。这种测试可以让你无视代码的依赖关系去测试代码的有效性。核心思想就是如果代码按设计正常工作，并且依赖关系也正常，那么他们应该会同时工作正常。

有些时候，我们代码所需要的依赖可能尚未开发完成，甚至还不存在，那如何让我们的开发进行下去呢？使用mock可以让开发进行下去，mock技术的目的和作用就是模拟一些在应用中不容易构造或者比较复杂的对象，从而把测试与测试边界以外的对象隔离开。

我们可以自己编写自定义的Mock对象实现mock技术，但是编写自定义的Mock对象需要额外的编码工作，同时也可能引入错误。现在实现mock技术的优秀开源框架有很多，Mockito就是一个优秀的用于单元测试的mock框架。Mockito已经在github上开源，详细请点击：https://github.com/mockito/mockito

除了Mockito以外，还有一些类似的框架，比如：

 1. EasyMock：早期比较流行的MocK测试框架。它提供对接口的模拟，能够通过录制、回放、检查三步来完成大体的测试过程，可以验证方法的调用种类、次数、顺序，可以令 Mock 对象返回指定的值或抛出指定异常
 2. PowerMock：这个工具是在EasyMock和Mockito上扩展出来的，目的是为了解决EasyMock和Mockito不能解决的问题，比如对static, final,private方法均不能mock。其实测试架构设计良好的代码，一般并不需要这些功能，但如果是在已有项目上增加单元测试，老代码有问题且不能改时，就不得不使用这些功能了
 3. JMockit：JMockit 是一个轻量级的mock框架是用以帮助开发人员编写测试程序的一组工具和API，该项目完全基于 Java 5 SE 的 java.lang.instrument 包开发，内部使用 ASM 库来修改Java的Bytecode

<!--more-->
### Mockito使用举例
#### 1，
```java

import org.junit.Test;
import org.mockito.InOrder;
import org.mockito.exceptions.verification.NoInteractionsWanted;

import java.util.LinkedList;
import java.util.List;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.mockito.Mockito.*;

@SuppressWarnings("unchecked")
public class _01_Mock {
    // Mock对象或接口
    @Test
    public void mock() {
        List<String> mockedList = mock(List.class);

        //using mock object
        mockedList.add("one");
        mockedList.clear();

        //verification
        verify(mockedList).add("one");
        verify(mockedList).clear();
    }
    // 打桩 
    @Test
    public void stubbing() {
        //You can mock concrete classes, not just interfaces
        LinkedList mockedList = mock(LinkedList.class);

        //stubbing
        when(mockedList.get(0)).thenReturn("first1", "first2");
        when(mockedList.get(1)).thenThrow(new RuntimeException("RuntimeException"));

        // 桩能够被重复打，调用的结果是按照打桩的顺序，如果调用次数大于打桩，则会多次返回最后一次打桩
        assertEquals(mockedList.get(0), "first1");
        assertEquals(mockedList.get(0), "first2");
        assertEquals(mockedList.get(0), "first2");
        //following throws runtime exception
        try {
            mockedList.get(1);
            assertFalse(true);// 不会执行
        } catch (RuntimeException e) {
            assertEquals(e.getMessage(), "RuntimeException");
        }
        // 没被打桩的数据返回null ,int/integer返回0，布尔返回false
        assertEquals(mockedList.get(999), null);
        //Although it is possible to verify a stubbed invocation, usually it's just redundant
        //If your code cares what get(0) returns, then something else breaks (often even before verify() gets executed).
        //If your code doesn't care what get(0) returns, then it should not be stubbed. Not convinced? See here.
        verify(mockedList, times(3)).get(0);
    }
    // 校验方法的执行次数
    @Test
    public void verifyingExactNumberOfInvocations() {
        LinkedList mockedList = mock(LinkedList.class);
        mockedList.add("once");

        mockedList.add("twice");
        mockedList.add("twice");

        mockedList.add("three times");
        mockedList.add("three times");
        mockedList.add("three times");

        //校验次数，默认使用了times(1)
        verify(mockedList).add("once");
        verify(mockedList, times(1)).add("once");

        //exact number of invocations verification
        verify(mockedList, times(2)).add("twice");
        verify(mockedList, times(3)).add("three times");

        //verification using never(). never() is an alias to times(0)
        verify(mockedList, never()).add("never happened");

        //verification using atLeast()/atMost()
        verify(mockedList, atLeastOnce()).add("three times");
        verify(mockedList, atLeast(2)).add("three times");
        verify(mockedList, atMost(5)).add("three times");
    }
    // 模拟抛出异常
    @Test(expected = RuntimeException.class)
    public void stubbingVoidMethodsWithExceptions() {
        LinkedList mockedList = mock(LinkedList.class);
        doThrow(new RuntimeException()).when(mockedList).clear();
        mockedList.clear();// 会抛出异常
    }

    // 校验方法的执行顺序
    @Test
    public void verificationInOrder() {
        // A. Single mock whose methods must be invoked in a particular order
        List singleMock = mock(List.class);
        singleMock.add("was added first");
        singleMock.add("was added second");
        //create an inOrder verifier for a single mock
        InOrder inOrder = inOrder(singleMock);

        //following will make sure that add is first called with "was added first, then with "was added second"
        inOrder.verify(singleMock).add("was added first");
        inOrder.verify(singleMock).add("was added second");

        // 多次Mock的顺序也能校验
        List firstMock = mock(List.class);
        List secondMock = mock(List.class);
        firstMock.add("was called first");
        secondMock.add("was called second");
        //create inOrder object passing any mocks that need to be verified in order
        InOrder inOrder2 = inOrder(firstMock, secondMock);
        //following will make sure that firstMock was called before secondMock
        inOrder2.verify(firstMock).add("was called first");
        inOrder2.verify(secondMock).add("was called second");
    }
    
    // 校验其他的mock对象从来没有过交互（interacted）
    @Test
    public void makingSureInteractionsNeverHappenedOnMock() {
        //using mocks - only mockOne is interacted
        List mockOne = mock(List.class);
        List mockTwo = mock(List.class);
        List mockThree = mock(List.class);
        mockOne.add("one");
        // 正常校验
        verify(mockOne).add("one");
        verify(mockOne, never()).add("two");

        // 校验其他的mock对象从来没有过交互（interacted）
        verifyZeroInteractions(mockTwo, mockThree);
    }
   //校验多余的调用,尽量不要使用这种校验，因为会让用例缺少维护性
    @Test(expected = NoInteractionsWanted.class)
    public void findingRedundantInvocations() {
        LinkedList mockedList = mock(LinkedList.class);
        //using mocks
        mockedList.add("one");
        mockedList.add("two");

        verify(mockedList).add("one");

        // 这个校验会抛出异常，因为后续还有交互
        verifyNoMoreInteractions(mockedList);
    }
}

```
#### 参数匹配
```java

import org.junit.Test;
import org.mockito.ArgumentMatcher;

import java.util.Arrays;
import java.util.Collection;
import java.util.LinkedList;
import java.util.List;

import static org.junit.Assert.assertEquals;
import static org.mockito.Mockito.*;

@SuppressWarnings("unchecked")
public class _02_ArgumentMatcher {
    @Test
    public void argumentMatcher1() {
        LinkedList<String> mockedList = mock(LinkedList.class);
        // 使用anyInt来匹配任意整数进行打桩
        when(mockedList.get(anyInt())).thenReturn("element");
        //following prints "element"
        assertEquals(mockedList.get(999), "element");
        //you can also verify using an argument matcher
        verify(mockedList).get(anyInt());
    }

    @Test
    public void argumentMatcher2() {
        MockObject mock = mock(MockObject.class);

        when(mock.dryRun(anyBoolean())).thenReturn("state");// stubbing using anyBoolean() argument matcher
        assertEquals(mock.dryRun(null), null); // 没有打桩，返回null

        when(mock.dryRun(isNull())).thenReturn("state"); // 为null打桩
        assertEquals(mock.dryRun(null), "state");// ok

        when(mock.dryRun(anyBoolean())).thenReturn("state"); // 还原
        assertEquals(mock.dryRun(true), "state");// ok

        // 如果有参数使用了ArgumentMatcher，所有的参数都要使用
        verify(mock, times(0)).someMethod(anyInt(), anyString(), eq("third argument"));
        // 错误做法
        // verify(mock,times(0)).someMethod(anyInt(), anyString(), "third argument");
    }

    @Test
    public void customerArgumentMatcher3() {
        List mock = mock(List.class);
        when(mock.addAll(argThat(new ListOfTwoElements()))).thenReturn(true);

        mock.addAll(Arrays.asList("one", "two"));

        verify(mock).addAll(argThat(new ListOfTwoElements()));

        //To keep it readable you can extract method, e.g:
        verify(mock).addAll(argThat(new ListOfTwoElements()));
        //becomes
        verify(mock).addAll(listOfTwoElements());
        verify(mock).addAll(argThat(list -> list.size() == 2));
    }

    private Collection listOfTwoElements() {
        return Arrays.asList(new ListOfTwoElements(), new ListOfTwoElements());
    }

    private class MockObject {
        String dryRun(Boolean b) {
            return String.valueOf(b);
        }

        void someMethod(int i, String s, String s2) {
        }
    }

    class ListOfTwoElements implements ArgumentMatcher<List> {
        @Override
        public boolean matches(List list) {
            return list.size() == 2;
        }

        @Override
        public String toString() {
            //printed in verification errors
            return "[list of 2 elements]";
        }
    }
}

```
#### spy

```java
import org.junit.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mockito;
import org.mockito.invocation.InvocationOnMock;
import org.mockito.stubbing.Answer;

import java.util.LinkedList;
import java.util.List;

import static org.junit.Assert.*;
import static org.mockito.Mockito.*;

@SuppressWarnings("unchecked")
//@RunWith(MockitoJUnitRunner.class)
public class _03_Spy {

    /**
     * spy()方法可以被用来封装 java 对象。被封装后，除非特殊声明（打桩 stub），否则都会真正的调用对象里面的每一个方法
     * 使用@Mock生成的类，所有方法都不是真实的方法，而且返回值都是NULL。
     * 使用@Spy生成的类，所有方法都是真实方法，返回值都是和真实方法一样的。
     * 用when去设置模拟返回值时，它里面的方法（dao.getOrder()）会先执行一次。
     * 使用doReturn去设置的话，就不会产生上面的问题，因为有when来进行控制要模拟的方法，所以不会执行原来的方法。
     */
    @Test
    public void testSpy() {
        List list = new LinkedList();
        List spy = spy(list);

        //预设size()期望值
        when(spy.size()).thenReturn(100);
        //using the spy calls *real* methods
        spy.add("one");
        spy.add("two");
        assertEquals("one", spy.get(0));
        assertEquals(100, spy.size());
        assertTrue(spy.contains("one"));

        //optionally, you can verify
        verify(spy).add("one");
        verify(spy).add("two");
    }


    /**
     * 区别
     * when(spy.get(0)).thenReturn("foo");
     * doReturn("foo").when(spy).get(0);
     */
    @Test
    public void differenceBetweenWhenAndDoReturnWhen() {
        List list = new LinkedList();
        List spy = spy(list);
        //下面预设的spy.get(0)会报错，因为会调用真实对象的get(0)，所以会抛出IndexOutOfBoundsException
        // when(spy.get(0)).thenReturn("foo");
        //必须使用doReturn() 来打桩
        doReturn("foo").when(spy).get(0);
        System.out.println("spy = " + spy.get(0));
    }

    /**
     * 改变你不打桩的调用的返回值
     * .Changing default return values of unstubbed invocations (Since 1.7)
     */
    @Test
    public void unstubbedInvocations() {
        Foo mock = mock(Foo.class);
        assertNull(mock.getStuff());
        Foo mock2 = mock(Foo.class, RETURNS_SMART_NULLS);
        assertEquals("", mock2.getStuff());
        Foo mock3 = mock(Foo.class, new Answer<String>() {
            @Override
            public String answer(InvocationOnMock invocation) throws Throwable {
                return "Custom String";
            }
        });
        assertEquals("Custom String", mock3.getStuff());
    }

    @Test
    public void unstubbedInvocation2s() {
        Foo mock = mock(Foo.class);
        ArgumentCaptor<Person> argument = ArgumentCaptor.forClass(Person.class);
        verify(mock).doSomething(argument.capture());
        assertEquals("John", argument.getValue().getName());
    }

    private class Foo {
        void doSomething(Person arg) {}
        public String getStuff() {
            return null;
        }
    }

    private class Person {
        String getName() {
            return "John";
        }
    }

    private class MockObject {
        String someMethod(String arg) {
            return arg;
        }
        void someVoidMethod() {}
    }
}

```
#### 注解方面
```java

import org.junit.Before;
import org.junit.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import org.mockito.stubbing.Answer;

import java.util.LinkedList;

import static org.junit.Assert.*;
import static org.mockito.Mockito.*;

@SuppressWarnings("unchecked")
//@RunWith(MockitoJUnitRunner.class)
public class _04_Annotation {
    @Mock
    private MockObject mock; // 需要使用@Before MockitoAnnotations.initMocks(this) 或者使用@RunWith(MockitoJUnitRunner.class)来Mock

    @Before
    public void before() {
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void initMockAnnotation() {
        assertNotNull(mock);
    }

    @Test
    public void stubbingConsecutiveCalls() { // 连续打桩，连续调用
        when(mock.someMethod("some arg")).thenReturn("foo").thenReturn("bar");
        when(mock.someMethod("some arg")).thenReturn("foo", "bar");//或者这样，方法两次还是会覆盖
        assertEquals(mock.someMethod("some arg"), "foo");
        assertEquals(mock.someMethod("some arg"), "bar");
        assertEquals(mock.someMethod("some arg"), "bar");
    }

    @Test
    public void stubbingWithCallbacks() { // 连续打桩，连续调用
        when(mock.someMethod(anyString())).thenAnswer((Answer<String>) invocation -> {
            // 在invocation中可以调用真实的方法，可以获取mock的对象，可以获得方法名，参数名
            // 最终能做的就是根据不同的参数，返回不能的Mock结果
            // 注意answer方法运行线程是在主线程，并不是异步调用
            System.out.println("Thread.currentThread().getName() = " + Thread.currentThread().getName());
            return "called with arguments: " + invocation.getArgument(0);
        });
        assertEquals(mock.someMethod("foo"), "called with arguments: foo");
    }

    /**
     * doReturn()|doThrow()| doAnswer()|doNothing()|doCallRealMethod() family of methods 用于
     * 1，打桩空方法 2，打桩spy的对象，3，对同一个对象重复打桩。或在测试过程中改变mock行为
     */
    @Test
    public void doMethodsFamily() {
        LinkedList mockedList = mock(LinkedList.class);
        // doThrow()
        doThrow(new RuntimeException()).when(mockedList).clear();
        try {
            mockedList.clear();
            fail();
        } catch (RuntimeException e) {
            assertTrue(true);
        }
        // doReturn()
        doReturn(false).when(mockedList).add("test");
        assertEquals(false, mockedList.add("test"));
        // doNothing()
        doNothing().when(mockedList).clear();
        verify(mockedList).clear();
        // doCallRealMethod()
        doCallRealMethod().when(mock).someVoidMethod();
        mock.someVoidMethod();//这会真正的调用someVoidMethod()
        // oAnswer(Answer)
    }

    private class MockObject {
        String someMethod(String arg) {
            return arg;
        }
        void someVoidMethod() {
        }
    }
}
```

#### 参考

 1. [Mockito 英文文档 )](http://static.javadoc.io/org.mockito/mockito-core/2.12.0/org/mockito/Mockito.html)
 2. [Mockito 中文文档 ( 2.0.26 beta )](http://www.devtf.cn/?p=1315)
 3. [Mockito使用指南](http://blog.csdn.net/shensky711/article/details/52771493#mock和mockito的关系)
 4. [Mockito2 WIKI](https://github.com/mockito/mockito/wiki/What%27s-new-in-Mockito-2)
 5. [Mockito官网](http://site.mockito.org/)
