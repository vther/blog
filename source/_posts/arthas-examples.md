#### 启动arthas-boot

在新的`Terminal 2`里，下载`arthas-boot.jar`，再用`java -jar`命令启动：

```
wget https://alibaba.github.io/arthas/arthas-boot.jar java -jar arthas-boot.jar --target-ip 0.0.0.0
```

`arthas-boot`是`Arthas`的启动程序，它启动后，会列出所有的Java进程，用户可以选择需要诊断的目标进程。

选择第一个进程，输入 `1` ，再`Enter/回车`：

Attach成功之后，会打印Arthas LOGO。输入 `help` 可以获取到更多的帮助信息。



#### 查看JVM信息

下面介绍Arthas里查看`JVM`信息的命令。

### sysprop

`sysprop` 可以打印所有的System Properties信息。

也可以指定单个key： `sysprop java.version`

也可以通过`grep`来过滤： `sysprop | grep user`

可以设置新的value： `sysprop testKey testValue`

### sysenv

`sysenv` 命令可以获取到环境变量。和`sysprop`命令类似。

### jvm

`jvm` 命令会打印出`JVM`的各种详细信息。

### dashboard

`dashboard` 命令可以查看当前系统的实时数据面板。

输入 `Q` 或者 `Ctrl+C` 可以退出dashboard命令。

#### thread

`thread 1` 命令会打印线程ID 1的栈。

Arthas支持管道，可以用 `thread 1 | grep 'main('` 查找到`main class`。

可以看到`main class`是`demo.MathGame`：

```
$ thread 1 | grep 'main('
    at demo.MathGame.main(MathGame.java:17)
```

#### watch

通过`watch`命令可以查看函数的参数/返回值/异常信息。

```
watch demo.MathGame primeFactors returnObj
```

输入 `Q` 或者 `Ctrl+C` 退出watch命令。

### help

Arthas里每一个命令都有详细的帮助信息。可以用`-h`来查看。帮助信息里有`EXAMPLES`和`WIKI`链接。

比如：

```
sysprop -h
```

### 自动补全

Arthas支持丰富的自动补全功能，在使用有疑惑时，可以输入`Tab`来获取更多信息。

比如输入 `sysprop java.` 之后，再输入`Tab`，会补全出对应的key：

```
$ sysprop java.
java.runtime.name             java.protocol.handler.pkgs    java.vm.version
java.vm.vendor                java.vendor.url               java.vm.name
...
```

### readline的快捷键支持

Arthas支持常见的命令行快捷键，比如`Ctrl + A`跳转行首，`Ctrl + E`跳转行尾。

更多的快捷键可以用 `keymap` 命令查看。

### 历史命令的补全

如果想再执行之前的命令，可以在输入一半时，按`Up/↑` 或者 `Ddown/↓`，来匹配到之前的命令。

比如之前执行过`sysprop java.version`，那么在输入`sysprop ja`之后，可以输入`Up/↑`，就会自动补全为`sysprop java.version`。

如果想查看所有的历史命令，也可以通过 `history` 命令查看到。

### pipeline

Arthas支持在pipeline之后，执行一些简单的命令，比如：

```
sysprop | grep java
sysprop | wc -l
```

#### sc/sm 查看已加载的类

下面介绍Arthas里查找已加载类的命令。

### sc

`sc` 命令可以查找到所有JVM已经加载到的类。

如果搜索的是接口，还会搜索所有的实现类。比如查看所有的`Filter`实现类：

```
sc javax.servlet.Filter
```

通过`-d`参数，可以打印出类加载的具体信息，很方便查找类加载问题。

```
sc -d javax.servlet.Filter
```

`sc`支持通配，比如搜索所有的`StringUtils`：

```
sc *StringUtils
```

### sm

`sm`命令则是查找类的具体函数。比如：

```
sm java.math.RoundingMode
```

通过`-d`参数可以打印函数的具体属性：

```
sm -d java.math.RoundingMode
```

也可以查找特定的函数，比如查找构造函数：

```
sm java.math.RoundingMode <init>
```

#### jad

可以通过 `jad` 命令来反编译代码：

```
jad com.example.demo.arthas.user.UserController
```

通过`--source-only`参数可以只打印出在反编译的源代码：

```
jad --source-only com.example.demo.arthas.user.UserController
```

#### ognl

在Arthas里，有一个单独的`ognl`命令，可以动态执行代码。

### 调用static函数

```
ognl '@java.lang.System@out.println("hello ognl")'
```

可以检查`Terminal 1`里的进程输出，可以发现打印出了`hello ognl`。

### 获取静态类的静态字段

获取`UserController`类里的`logger`字段：

```
ognl -c 1be6f5c3 @com.example.demo.arthas.user.UserController@logger
```

还可以通过`-x`参数控制返回值的展开层数。比如：

```
ognl -c 1be6f5c3 -x 2 @com.example.demo.arthas.user.UserController@logger
```

### 执行多行表达式，赋值给临时变量，返回一个List

```
ognl '#value1=@System@getProperty("java.home"), #value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
$ ognl '#value1=@System@getProperty("java.home"), #value2=@System@getProperty("java.runtime.name"), {#value1, #value2}'
@ArrayList[
    @String[/Library/Java/JavaVirtualMachines/jdk1.8.0_162.jdk/Contents/Home/jre],
    @String[Java(TM) SE Runtime Environment],
]
```

### 更多

在Arthas里`ognl`表达式是很重要的功能，在很多命令里都可以使用`ognl`表达式。

一些更复杂的用法，可以参考：

- OGNL特殊用法请参考：<https://github.com/alibaba/arthas/issues/71>
- OGNL表达式官方指南：<https://commons.apache.org/proper/commons-ognl/language-guide.html>



#### Exit/Shutdown

## 退出Arthas

用 `exit` 或者 `quit` 命令可以退出Arthas。

退出Arthas之后，还可以再次用 `java -jar arthas-boot.jar` 来连接。

## 彻底退出Arthas

`exit/quit`命令只是退出当前session，arthas server还在目标进程中运行。

想完全退出Arthas，可以执行 `shutdown` 命令。



#### 案例: 排查函数调用异常

### 现象

目前，访问 <http://localhost/user/0> ，会返回500异常：

```
curl http://localhost/user/0
{"timestamp":1550223186170,"status":500,"error":"Internal Server Error","exception":"java.lang.IllegalArgumentException","message":"id < 1","path":"/user/0"}
```

但请求的具体参数，异常栈是什么呢？

### 查看UserController的 参数/异常

在Arthas里执行：

```
watch com.example.demo.arthas.user.UserController * '{params, throwExp}'
```

1. 第一个参数是类名，支持通配
2. 第二个参数是函数名，支持通配

访问 `curl http://localhost/user/0` ,`watch`命令会打印调用的参数和异常

```bash
$ watch com.example.demo.arthas.user.UserController * '{params, throwExp}'
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:2) cost in 53 ms.
ts=2019-02-15 01:35:25; [cost=0.996655ms] result=@ArrayList[
    @Object[][isEmpty=false;size=1],
    @IllegalArgumentException[java.lang.IllegalArgumentException: id < 1],
]
```

可以看到实际抛出的异常是`IllegalArgumentException`。

可以输入 `Q` 或者 `Ctrl+C` 退出watch命令。

如果想把获取到的结果展开，可以用`-x`参数：

```
watch com.example.demo.arthas.user.UserController * '{params, throwExp}' -x 2
```

### 返回值表达式

在上面的例子里，第三个参数是`返回值表达式`，它实际上是一个`ognl`表达式，它支持一些内置对象：

- loader
- clazz
- method
- target
- params
- returnObj
- throwExp
- isBefore
- isThrow
- isReturn

你可以利用这些内置对象来组成不同的表达式。比如返回一个数组：

```
watch com.example.demo.arthas.user.UserController * '{params[0], target, returnObj}'
```

更多参考： <https://alibaba.github.io/arthas/advice-class.html>

### 条件表达式

`watch`命令支持在第4个参数里写条件表达式，比如：

```
watch com.example.demo.arthas.user.UserController * returnObj 'params[0] > 100'
```

当访问 <https://2886795302-80-cykoria03.environments.katacoda.com/user/1>时，`watch`命令没有输出

当访问 <https://2886795302-80-cykoria03.environments.katacoda.com/user/101>时，`watch`会打印出结果。

```bash
$ watch com.example.demo.arthas.user.UserController * returnObj 'params[0] > 100'
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:2) cost in 47 ms.
ts=2019-02-13 19:42:12; [cost=0.821443ms] result=@User[
    id=@Integer[101],
    name=@String[name101],
]
```

### 当异常时捕获

`watch`命令支持`-e`选项，表示只捕获抛出异常时的请求：

```
watch com.example.demo.arthas.user.UserController * "{params[0],throwExp}" -e
```

### 按照耗时进行过滤

watch命令支持按请求耗时进行过滤，比如：

```
watch com.example.demo.arthas.user.UserController * '{params, returnObj}' '#cost>200'
```



#### 案例: 热更新代码

下面介绍通过`jad`/`mc`/`redefine` 命令实现动态更新代码的功能。

目前，访问 <http://localhost/user/0> ，会返回500异常：

```
curl http://localhost/user/0
{"timestamp":1550223186170,"status":500,"error":"Internal Server Error","exception":"java.lang.IllegalArgumentException","message":"id < 1","path":"/user/0"}
```

下面通过热更新代码，修改这个逻辑。

### jad反编译UserController

```
jad --source-only com.example.demo.arthas.user.UserController > /tmp/UserController.java
```

jad反编译的结果保存在 `/tmp/UserController.java`文件里了。

再打开一个`Terminal 3`，然后用vim来编辑`/tmp/UserController.java`：

```
vim /tmp/UserController.java
```

比如当 user id 小于1时，也正常返回，不抛出异常：

```java
    @GetMapping(value={"/user/{id}"})
    public User findUserById(@PathVariable Integer id) {
        logger.info("id: {}", (Object)id);
        if (id != null && id < 1) {
            return new User(id, "name" + id);
            // throw new IllegalArgumentException("id < 1");
        }
        return new User(id.intValue(), "name" + id);
    }
```

### sc查找加载UserController的ClassLoader

```
sc -d *UserController | grep classLoaderHash
$ sc -d *UserController | grep classLoaderHash
 classLoaderHash   1be6f5c3
```

可以发现是 spring boot `LaunchedURLClassLoader@1be6f5c3` 加载的。

### mc

保存好`/tmp/UserController.java`之后，使用`mc`(Memory Compiler)命令来编译，并且通过`-c`参数指定ClassLoader：

```
mc -c 1be6f5c3 /tmp/UserController.java -d /tmp
$ mc -c 1be6f5c3 /tmp/UserController.java -d /tmp
Memory compiler output:
/tmp/com/example/demo/arthas/user/UserController.class
Affect(row-cnt:1) cost in 346 ms
```

### redefine

再使用`redefine`命令重新加载新编译好的`UserController.class`：

```
redefine /tmp/com/example/demo/arthas/user/UserController.class
$ redefine /tmp/com/example/demo/arthas/user/UserController.class
redefine success, size: 1
```

### 热修改代码结果

`redefine`成功之后，再次访问 <https://2886795302-80-cykoria03.environments.katacoda.com/user/0> ，结果是：

```
{
  "id": 0,
  "name": "name0"
}
```



#### 案例: 动态更新应用Logger Level

在这个案例里，动态修改应用的Logger Level。

### 查找UserController的ClassLoader

```
sc -d com.example.demo.arthas.user.UserController | grep classLoaderHash
$ sc -d com.example.demo.arthas.user.UserController | grep classLoaderHash
 classLoaderHash   1be6f5c3
```

### 用ognl获取logger

```
ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'
$ ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'
@Logger[
    serialVersionUID=@Long[5454405123156820674],
    FQCN=@String[ch.qos.logback.classic.Logger],
    name=@String[com.example.demo.arthas.user.UserController],
    level=null,
    effectiveLevelInt=@Integer[20000],
    parent=@Logger[Logger[com.example.demo.arthas.user]],
    childrenList=null,
    aai=null,
    additive=@Boolean[true],
    loggerContext=@LoggerContext[ch.qos.logback.classic.LoggerContext[default]],
]
```

可以知道`UserController@logger`实际使用的是logback。可以看到`level=null`，则说明实际最终的level是从`root` logger里来的。

### 单独设置UserController的logger level

```
ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger.setLevel(@ch.qos.logback.classic.Level@DEBUG)'
```

再次获取`UserController@logger`，可以发现已经是`DEBUG`了：

```
ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'
$ ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'
@Logger[
    serialVersionUID=@Long[5454405123156820674],
    FQCN=@String[ch.qos.logback.classic.Logger],
    name=@String[com.example.demo.arthas.user.UserController],
    level=@Level[DEBUG],
    effectiveLevelInt=@Integer[10000],
    parent=@Logger[Logger[com.example.demo.arthas.user]],
    childrenList=null,
    aai=null,
    additive=@Boolean[true],
    loggerContext=@LoggerContext[ch.qos.logback.classic.LoggerContext[default]],
]
```

### 修改logback的全局logger level

通过获取`root` logger，可以修改全局的logger level：

```
ognl -c 1be6f5c3 '@org.slf4j.LoggerFactory@getLogger("root").setLevel(@ch.qos.logback.classic.Level@DEBUG)'
```

#### 案例: 排查logger冲突问题

在这个案例里，展示排查logger冲突的方法。

### 确认应用使用的logger系统

以`UserController`为例，它使用的是slf4j api，但实际使用到的logger系统是logback。

```
ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'
$ ognl -c 1be6f5c3 '@com.example.demo.arthas.user.UserController@logger'
@Logger[
    serialVersionUID=@Long[5454405123156820674],
    FQCN=@String[ch.qos.logback.classic.Logger],
    name=@String[com.example.demo.arthas.user.UserController],
    level=null,
    effectiveLevelInt=@Integer[20000],
    parent=@Logger[Logger[com.example.demo.arthas.user]],
    childrenList=null,
    aai=null,
    additive=@Boolean[true],
    loggerContext=@LoggerContext[ch.qos.logback.classic.LoggerContext[default]],
]
```

### 获取logback实际加载的配置文件

```
ognl -c 1be6f5c3 '#map1=@org.slf4j.LoggerFactory@getLogger("root").loggerContext.objectMap, #map1.get("CONFIGURATION_WATCH_LIST")'
```

### 使用classloader命令查找可能存在的logger配置文件

```
classloader -c 1be6f5c3 -r logback-spring.xml
$ classloader -c 1be6f5c3 -r logback-spring.xml
 jar:file:/Users/hengyunabc/code/java/spring-boot-inside/demo-arthas-spring-boot/target/demo-arthas-spring-boot-0.0.1-SNAPSHOT.jar!/BOOT-INF/classes!/logback-spring.xml

Affect(row-cnt:1) cost in 13 ms.
```

可以知道加载的配置的具体来源。

可以尝试加载容易冲突的文件：

```
classloader -c 1be6f5c3 -r logback.xml
classloader -c 1be6f5c3 -r log4j.properties
```

#### 案例: 获取Spring Context

在这个案例里，展示获取spring context，再获取bean，然后调用函数。

### 使用tt命令获取到spring context

`tt`即 TimeTunnel，它可以记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测。

- <https://alibaba.github.io/arthas/tt.html>

```
tt -t org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter invokeHandlerMethod
```

访问：<https://2886795302-80-cykoria03.environments.katacoda.com/user/1>

可以看到`tt`命令捕获到了一个请求：

```bash
$ tt -t org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdaptePress Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 252 ms.
 INDE  TIMESTAMP    COST(  IS-R  IS-  OBJECT     CLASS               METHOD
 X                  ms)    ET    EXP
-----------------------------------------------------------------------------------------
 1000  2019-02-15   4.583  true  fal  0xc93cf1a  RequestMappingHand  invokeHandlerMethod
       15:38:32     923          se              lerAdapter
```

### 使用tt命令从调用记录里获取到spring context

输入 `Q` 或者 `Ctrl + C` 退出上面的 `tt -t`命令。

```
tt -i 1000 -w 'target.getApplicationContext()'
$ tt -i 1000 -w 'target.getApplicationContext()'
@AnnotationConfigEmbeddedWebApplicationContext[
    reader=@AnnotatedBeanDefinitionReader[org.springframework.context.annotation.AnnotatedBeanDefinitionReader@2e457641],
    scanner=@ClassPathBeanDefinitionScanner[org.springframework.context.annotation.ClassPathBeanDefinitionScanner@6eb38026],
    annotatedClasses=null,
    basePackages=null,
]
Affect(row-cnt:1) cost in 439 ms.
```

## 获取spring bean，并调用函数

```
tt -i 1000 -w 'target.getApplicationContext().getBean("helloWorldService").getHelloMessage()'
```

结果是：

```
bash $ tt -i 1000 -w 'target.getApplicationContext().getBean("helloWorldService").getHelloMessage()' @String[Hello World] Affect(row-cnt:1) cost in 52 ms.
```



#### 案例: 排查HTTP请求返回401

在这个案例里，展示排查HTTP 401问题的技巧。

访问： <https://2886795302-80-cykoria03.environments.katacoda.com/admin>

结果是：

```
Something went wrong: 401 Unauthorized
```

我们知道`401`通常是被权限管理的`Filter`拦截了，那么到底是哪个`Filter`处理了这个请求，返回了401？

### 跟踪所有的Filter函数

开始trace：

```
trace javax.servlet.Filter *
```

访问： <https://2886795302-80-cykoria03.environments.katacoda.com/admin>

可以在调用树的最深层，找到`AdminFilterConfig$AdminFilter`返回了`401`：

```
+---[3.806273ms] javax.servlet.FilterChain:doFilter()
|   `---[3.447472ms] com.example.demo.arthas.AdminFilterConfig$AdminFilter:doFilter()
|       `---[0.17259ms] javax.servlet.http.HttpServletResponse:sendError()
```

### 通过stack获取调用栈

上面是通过`trace`命令来获取信息，从结果里，我们可以知道通过`stack`跟踪`HttpServletResponse:sendError()`，同样可以知道是哪个`Filter`返回了`401`

执行：

```
stack javax.servlet.http.HttpServletResponse sendError 'params[0]==401'
```

访问： <https://2886795302-80-cykoria03.environments.katacoda.com/admin>

```bash
$ stack javax.servlet.http.HttpServletResponse sendError 'params[0]==401'
Press Q or Ctrl+C to abort.
Affect(class-cnt:2 , method-cnt:4) cost in 87 ms.
ts=2019-02-15 16:44:06;thread_name=http-nio-8080-exec-6;id=16;is_daemon=true;priority=5;TCCL=org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedWebappClassLoader@8546cd5
    @org.apache.catalina.connector.ResponseFacade.sendError()
        at com.example.demo.arthas.AdminFilterConfig$AdminFilter.doFilter(AdminFilterConfig.java:38)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
        at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
```

#### 案例: 排查HTTP请求返回404

在这个案例里，展示排查HTTP 404问题的技巧。

访问： <https://2886795302-80-cykoria03.environments.katacoda.com/a.txt>

结果是：

```
Something went wrong: 404 Not Found
```

那么到底是哪个Servlet处理了这个请求，返回了404？

### 跟踪所有的Servlet函数

开始trace：

```
trace javax.servlet.Servlet * > /tmp/servlet.txt
```

访问： <https://2886795302-80-cykoria03.environments.katacoda.com/a.txt>

在`Terminal 3`里，查看`/tmp/servlet.txt`的内容：

```
less /tmp/servlet.txt
```

`/tmp/servlet.txt`里的内容会比较多，需要耐心找到调用树里最长的地方。

可以发现请求最终是被`freemarker`处理的：

```
`---[13.974188ms] org.springframework.web.servlet.ViewResolver:resolveViewName()     +---[0.045561ms] javax.servlet.GenericServlet:<init>()     +---[min=0.045545ms,max=0.074342ms,total=0.119887ms,count=2] org.springframework.web.servlet.view.freemarker.FreeMarkerView$GenericServletAdapter:<init>()     +---[0.170895ms] javax.servlet.GenericServlet:init()     |   `---[0.068578ms] javax.servlet.GenericServlet:init()     |       `---[0.021793ms] javax.servlet.GenericServlet:init()     `---[0.164035ms] javax.servlet.GenericServlet:getServletContext()
```

#### 案例: 理解Spring Boot应用的ClassLoader结构

下面介绍`classloader`命令的功能。

先访问一个jsp网页，触发jsp的加载： <https://2886795302-80-cykoria03.environments.katacoda.com/hello>

### 列出所有ClassLoader

```
classloader -l
$ classloader -l
 name                                                             loadedCount  hash      parent
 BootstrapClassLoader                                             2724         null      null
 com.taobao.arthas.agent.ArthasClassloader@411ce1ab               2009         411ce1ab  sun.misc.Launcher$ExtClassLoader@7494e528
 com.taobao.arthas.agent.ArthasClassloader@22ae1234               1253         22ae1234  sun.misc.Launcher$ExtClassLoader@7494e528
 org.apache.jasper.servlet.JasperLoader@65361d9a                  1            65361d9a  TomcatEmbeddedWebappClassLoader
                                                                                           context: ROOT
                                                                                           delegate: true
                                                                                         ----------> Parent Classloader:
                                                                                         org.springframework.boot.loader.LaunchedURLClassLoader@1be6f5c3

 TomcatEmbeddedWebappClassLoader                                  0            8546cd5   org.springframework.boot.loader.LaunchedURLClassLoader@1be6f5c3
   context: ROOT
   delegate: true
 ----------> Parent Classloader:
 org.springframework.boot.loader.LaunchedURLClassLoader@1be6f5c3

 org.springframework.boot.loader.LaunchedURLClassLoader@1be6f5c3  5416         1be6f5c3  sun.misc.Launcher$AppClassLoader@3d4eac69
 sun.misc.Launcher$AppClassLoader@3d4eac69                        45           3d4eac69  sun.misc.Launcher$ExtClassLoader@7494e528
 sun.misc.Launcher$ExtClassLoader@7494e528                        4            7494e528  null
```

- TomcatEmbeddedWebappClassLoader 加载的class数量是0，所以在spring boot embedded tomcat里，它只是一个空壳，所有的类加载都是`LaunchedURLClassLoader`完成的

### 列出ClassLoader里加载的所有类

列出上面的`org.apache.jasper.servlet.JasperLoader`加载的类：

```
classloader -a -c 65361d9a
$ classloader -a -c 65361d9a
 hash:1698045338, org.apache.jasper.servlet.JasperLoader@65361d9a
 org.apache.jsp.jsp.hello_jsp
```

### 反编译jsp的代码

```
jad org.apache.jsp.jsp.hello_jsp
$ jad org.apache.jsp.jsp.hello_jsp

ClassLoader:
+-org.apache.jasper.servlet.JasperLoader@65361d9a
  +-TomcatEmbeddedWebappClassLoader
      context: ROOT
...
```

### 查看ClassLoader树

```
classloader -t
$ classloader -t
+-BootstrapClassLoader
+-sun.misc.Launcher$ExtClassLoader@28cbbddd
  +-com.taobao.arthas.agent.ArthasClassloader@8c25e55
  +-sun.misc.Launcher$AppClassLoader@55f96302
    +-org.springframework.boot.loader.LaunchedURLClassLoader@1be6f5c3
      +-TomcatEmbeddedWebappClassLoader
          context: ROOT
          delegate: true
        ----------> Parent Classloader:
        org.springframework.boot.loader.LaunchedURLClassLoader@1be6f5c3

        +-org.apache.jasper.servlet.JasperLoader@21ae0fe2
```

### 列出ClassLoader的urls

比如上面查看到的spring LaunchedURLClassLoader的 hashcode是`1be6f5c3`，可以通过`-c`参数来列出它的所有urls：

```
classloader -c 1be6f5c3
$ classloader -c 1be6f5c3
jar:file:/home/scrapbook/tutorial/demo-arthas-spring-boot.jar!/BOOT-INF/classes!/
jar:file:/home/scrapbook/tutorial/demo-arthas-spring-boot.jar!/BOOT-INF/lib/spring-boot-starter-aop-1.5
.13.RELEASE.jar!/
...
```

### 加载指定ClassLoader里的资源文件

查找指定的资源文件： `classloader -c 1be6f5c3 -r logback-spring.xml`

```
$ classloader -c 1be6f5c3 -r logback-spring.xml
 jar:file:/home/scrapbook/tutorial/demo-arthas-spring-boot.jar!/BOOT-INF/classes!/logback-spring.xml
```

### 尝试加载指定的类

比如用上面的spring LaunchedURLClassLoader 尝试加载 `java.lang.String` ：

```
classloader -c 1be6f5c3 --load java.lang.String
$ classloader -c 1be6f5c3 --load java.lang.String
load class success.
 class-info        java.lang.String
 code-source
 name              java.lang.String
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       String
 modifier          final,public
 annotation
 interfaces        java.io.Serializable,java.lang.Comparable,java.lang.CharSequence
 super-class       +-java.lang.Object
 class-loader
 classLoaderHash   null
```

#### 案例：查找Top N线程

### 查看所有线程信息

```
thread
```

### 查看具体线程的栈

查看线程ID 16的栈：

```
thread 16
```

### 查看CPU使用率top n线程的栈

```
thread -n 3
```

查看5秒内的CPU使用率top n线程栈

```
thread -n 3 -i 5000
```

### 查找线程是否有阻塞

```
thread -b
```

#### Web Console

Arthas支持通过Web Socket来连接。

## 教程里的Web Console

<http://2886795302-8563-cykoria03.environments.katacoda.com/?ip=2886795302-8563-cykoria03.environments.katacoda.com&port=80>

> 注意：教程里访问的是80端口，因为做了端口转发。在本地体验时，需要访问8563端口。

## 本地体验

当在本地启动时，可以访问 <http://127.0.0.1:8563/> ，通过浏览器来使用Arthas。

![Arthas WebConsole](https://katacoda.com/hengyunabc/scenarios/arthas-advanced-cn/assets/web-console.png)

推荐通过“快速入门”来体验： <https://alibaba.github.io/arthas/quick-start.html>

#### Exit/Shutdown

## reset

Arthas在 watch/trace 等命令时，实际上是修改了应用的字节码，插入增强的代码。显式执行 `reset` 命令，可以清除掉这些增强代码。

## 退出Arthas

用 `exit` 或者 `quit` 命令可以退出Arthas。

退出Arthas之后，还可以再次用 `java -jar arthas-boot.jar` 来连接。

## 彻底退出Arthas

`exit/quit`命令只是退出当前session，arthas server还在目标进程中运行。

想完全退出Arthas，可以执行 `shutdown` 命令。

#### arthas-boot支持的参数

`arthas-boot.jar` 支持很多参数，可以执行 `java -jar arthas-boot.jar -h`来查看。

## 允许外部访问

默认情况下， arthas server侦听的是 `127.0.0.1` 这个IP，如果希望远程可以访问，可以使用`--target-ip`的参数。

```
java -jar arthas-boot.jar --target-ip
```

## 列出所有的版本

```
java -jar arthas-boot.jar --versions
```

使用指定版本：

```
java -jar arthas-boot.jar --use-version 3.1.0
```

## 只侦听Telnet端口，不侦听HTTP端口

```
java -jar arthas-boot.jar --telnet-port 9999 --http-port -1
```

## 打印运行的详情

```
java -jar arthas-boot.jar -v
```
