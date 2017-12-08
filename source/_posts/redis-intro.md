---
title: Redis的安装、和SpringBoot的集成
date: 2017年12月8日21:19:59
thumbnail: http://static.open-open.com/news/uploadImg/20160618/20160618093614_122.png
tags: 
 - Centos
 - VMware
 - Redis
categories: 
 - Redis
---
Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，内置了 复制，LUA脚本，LRU驱动事件，事务和不同级别的磁盘持久化， 并通过哨兵和自分区提供高可用性。

**本文为Redis初体验，主要包括以下内容**：

1，在Centos中安装Redis
2，Redis的简单使用
3，构建Spring Boot Redis工程
4，遇到问题记录

<!--more-->

----------
#### **Centos安装**
在Window上利用VMware虚拟机安装CentOS可以参考之前的一篇博客的部分内容，[在Window下安装CentOS](https://vther.coding.me/nginx-install-on-centos-in-windows/)
#### **Redis安装**

```bash
wget http://download.redis.io/releases/redis-3.2.11.tar.gz
tar xzf redis-3.2.11.tar.gz
cd redis-3.2.11
make
make test #可能需要 yum install tcl
make install 
```
redis基本指令，更多指令可以通过`{cmd} --help`来查看，[官网帮助文档](https://redis.io/topics/rediscli)
```
# 服务端,指定配置文、端口、日志级别
./redis-server /etc/redis.conf --port 7777 --loglevel verbose 
# 客户端,指定IP、端口、密码
./redis-cli -h 127.0.0.1 -p 6379 -a xxxxxx
```
#### **构建Spring Boot Redis工程**
1，使用http://start.spring.io/ 来生成redis工程，或者直接在SpringBoot中引入
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
2，[在application.properties配置Redis](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
```
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=192.168.1.105
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=123456
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
# 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
# 连接超时时间（毫秒）
spring.redis.timeout=0
```
3，添加测试代码
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class ApplicationTests {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
    @Test
    public void testConnection() {
        // 保存字符串
        stringRedisTemplate.opsForValue().set("foo", "bar");
        Assert.assertEquals("bar", stringRedisTemplate.opsForValue().get("foo"));
    }
}
```
#### **遇到问题**
1，Windows无法使用telnet
> **启用telnet：**控制面板 --> 程序 --> 启用或关闭Windows功能 --> Telnet客户端 --> 确定 --> `telnet 192.168.1.105 6379`

2，telnet不通或代码报错：*redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool*
> **打开6379端口：**`firewall-cmd --zone=public --add-port=6379/tcp --permanent`
> **或关闭linux防火墙：**`systemctl stop firewalld.service` （可选）禁止firewall开机启动`systemctl disable firewalld.service` 

3，代码报错：*DENIED Redis is running in protected mode because protected mode is enabled, no bind address was specified, no authentication password is requested to clients. *
> **Spring在异常中给了四种解决方案，任何一种都能解决：**
> 1) 使用`CONFIG SET protected-mode no` 使用 `CONFIG REWRITE` 可以让配置永久生效
> 2) 修改redis.conf，`protected-mode no`，重启Redis 
> 3) 重启Redis并带着`--protected-mode no` 选项  
> 4) 配置绑定地址`bind 192.168.1.104`或配置密码`requirepass xxxxxx`

#### **参考**
1，[Redis官网](https://redis.io)
2，[Redis中文网](http://www.redis.cn/)
3，[Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/1.6.2.RELEASE/reference/html/)
4，[Spring Boot Redis配置](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)
5，[CentOS 7.0关闭默认防火墙启用iptables防火墙](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/)