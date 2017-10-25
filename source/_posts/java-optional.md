---
title: Java8 之 Optional
date: 2017年10月25日21:19:59
thumbnail: http://www.runoob.com/wp-content/uploads/2013/12/java.jpg
tags: 
- optional
- JDK
- Java8
categories: 
- Java
---
在Java程序开发中大家经常会使用null，但是使用null会带来理论和实际操作上的种种问题。
>  它是错误之源
NullPointerException是目前Java程序开发中最典型的异常。
>  它会使你的代码膨胀。
它让你的代码充斥着深度嵌套的null检查，代码的可读性糟糕透顶。
>  它自身是毫无意义的。
null自身没有任何的语义，尤其是，它代表的是在静态类型语言中以一种错误的方式对
缺失变量值的建模。
>  它破坏了Java的哲学。
Java一直试图避免让程序员意识到指针的存在，唯一的例外是：null指针。
>  它在Java的类型系统上开了个口子。
null并不属于任何类型，这意味着它可以被赋值给任意引用类型的变量。这会导致问题，
原因是当这个变量被传递到系统中的另一个部分后，你将无法获知这个null变量最初的
赋值到底是什么类型。

<!--more-->
例如如下模型：
```java
    public class Person {
        private Car car;
        public Car getCar() {
            return car;
        }
    }
    public class Car {
        private Insurance insurance;
        public Insurance getInsurance() {
            return insurance;
        }
    }
    public class Insurance {
        private String name;
        public String getName() {
            return name;
        }
    }
```
如果你想通过Person对象获取他的车的保险的名字，你可能想象这么做：
```java
    public String getCarInsuranceName(Person person) {
        return person.getCar().getInsurance().getName();
    }
```
但是这样做很容易产生NullPointerException，如何避免呢？通常，你可以在需要的地方添
加null的检查，例如：
```java
    public String getCarInsuranceName(Person person) {
        if (person != null) {
            Car car = person.getCar();
            if (car != null) {
                Insurance insurance = car.getInsurance();
                if (insurance != null) {
                    return insurance.getName();
                }
            }
        }
        return "Unknown";
    }
```
但是这样的代码有以下缺点：扩展性差、可读性差、代码“质量”低。
下面引入我们今天的主角Optional，改良以后的代码如下：
```java
    public String getCarInsuranceName(Optional<Person> person) {
        return person.map(Person::getCar)
                .map(Car::getInsurance)
                .map(Insurance::getName)
                .orElse("Unknown");
    }
```
是不是看起来很清晰明了，心动不如行动，下面开始吧！

----------

# **Optional**

JDK DOCS 如下描述
> A container object which may or may not contain a non-null value. If a value is present, isPresent() will return true and get() will return the value.
> Additional methods that depend on the presence or absence of a contained value are provided, such as orElse() (return a default value if value not present) and ifPresent() (execute a block of code if the value is present).
> This is a value-based class; use of identity-sensitive operations (including reference equality (==), identity hash code, or synchronization) on instances of Optional may have unpredictable results and should be avoided.

> 这是一个可以包含null或者非null值的容器对象。如果值存在（不为null）则isPresent()方法会返回true，调用get()方法会返回容器中包含的对象。
> 其余的方法，依赖于容器中提供的值是否再存，例如orElse()（如果不存在将会返回一个默认值）以及 ifPresent() (如果存在则执行一段代码).
> Optional类是 [value-based class](http://docs.oracle.com/javase/8/docs/api/java/lang/doc-files/ValueBased.html)，实际使用中应该避免使用 == hashcode synchronize

更多API可以参考
[Optional Docs](http://docs.oracle.com/javase/8/docs/api/java/util/Optional.html)

## 1，基本用法

```java
    //调用工厂方法创建Optional实例
    Optional<String> name = Optional.of("Sanaulla");
    //下面创建了一个可能为空的Optional实例，例如，值为'null'
    Optional empty = Optional.ofNullable(null);
    //isPresent方法用来检查Optional实例中是否包含值
    if (name.isPresent()) {
      //在Optional实例内调用get()返回已存在的值
      System.out.println(name.get());//输出Sanaulla
    }
```

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

                            
***flatMap***：用法和map类似，但是参数需要提供使用Optional包裹的Function，调用后，返回的是没有经过Optional包裹的结果
                            
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