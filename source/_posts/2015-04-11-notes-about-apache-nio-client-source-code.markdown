---
layout: post
title: "Apache nio client 源码阅读笔记"
date: 2014-12-18 15:34:29 +0800
comments: true
categories: 
- java
- sourcecode
- nio
---

## 1. 部分主要类型功能简介

**`org.apache.http.impl.nio.client.HttpAsyncClients`**

它是一个比较接近业务层的介面, 提供一系列工厂方法, 用于构造 启动对 `HttpAsyncClient` 的构造和配置工作


**`org.apache.http.impl.nio.client.HttpAsyncClientBuilder`**

工厂方法创建buidler 用于构造具体的client.

从属性可以看出来, 使用策略模式, 支持各种类型策略的注入, 线程工厂, http接受/处理回调函数等等, cookie/auth 等http协议的内容也在这个地方处理.

它的 `build` 方法生成  `org.apache.http.impl.nio.client.CloseableHttpAsyncClient` 的子类型

**`org.apache.http.impl.nio.client.InternalHttpAsyncClient`**

默认的 HttpAsyncClient , 使用之前需要调用 start 方法.

http 请求任务通过 execute 方法提交, 其实现原理是
生成一个fureture对象用作和用户沟通,
再生成一个 `org.apache.http.impl.nio.client.DefaultClientExchangeHandlerImpl` 实例, 由它启动和 io loop 的数据交换
最后把 future 对象返回给用户层

**`org.apache.http.impl.nio.client.DefaultClientExchangeHandlerImpl`**

InternalHttpAsyncClient 会默认调用他的start方法.

在 start() 中, exec.prepare 负责向 stat 中填充数据和 request 配置

requestConnection() 向 connection manager 发起 connection , 并向他注册了一个 future callback, 意味着, conn manager 会向 ExchangeHandler 回调事件, 从而间接地向用户层发起回调.

**`org.apache.http.nio.conn.NHttpClientConnectionManager`**

负责接管 client 提交的http请求

关于接受请求的方法签名如下

```java
    Future<NHttpClientConnection> requestConnection(
            HttpRoute route,
            Object state,
            long connectTimeout,
            long connectionRequestTimeout,
            TimeUnit timeUnit,
            FutureCallback<NHttpClientConnection> callback);
```

HttpRoute 和请求的 ip 相关, 有一个相关的参数 MaxRequestPerRoute, 意思就是对同一个主机发起的最大连接数
connectTimeout tcp 等待建立 connection 的等待时间, 对方主机的tcp通道建立需要一定的时间
connectionRequestTimeout 表示排队等待 connection 的时间具体参考下文的 lease time

    ps1: connection 可以认为是一个保持着的长连接资源, 也就是 tcp 通道
    ps2: route 里面如果配置了 proxyhost(http代理), 那么这时候就起作用了

释放请求 releaseConnection, 这是真的将一个 connection 实例从 pool里拿出来, 中止链接, 回收实例

所以它完成的事情也不多, 就是向 pool 里注册和释放 connection

**`org.apache.http.impl.nio.conn.CPoolProxy`**

上文提到的 pool , 连接池

通过 lease 方法注册新的请求
首先 创建一个 org.apache.http.nio.pool.LeaseRequest#LeaseRequest 实例, 持有传入的参数

上文提到的 connectionRequestTimeout , 这里叫 lease time , 也就是 lease request 的 dealine

1. 在 processPendingRequest 中, 检查deadline ,如果到了, requestTimeout 触发, 触发TimeoutException (注意类型)
	* 也就是说, lease request 创建完毕, 发现已经到dealine ,那么就不向 pool 索取 connection 了, 直接退出
2. 接着向 routeToPool 和 route 对应的 connection pool (org.apache.http.nio.pool.RouteSpecificPool)  , 如果没有的话, 发起一个新的pool.
3. 接着从 pool 中索取一个 connection (org.apache.http.nio.reactor.SessionRequest) , 如果没有, 创建新的 pool , 将 socketTimeout 配置进去

**创建新的connection , 需要检查route对应的connection数量是否达到上限.**

对于拿到的新的 connection, 将上文所提到的 requestTimeout 作为 connection 的 connectionTimeout.

**`org.apache.http.impl.nio.reactor.DefaultConnectingIOReactor`**

nio 反应堆的默认实现.

继承于 org.apache.http.impl.nio.reactor.AbstractMultiworkerIOReactor , 负责发起worker线程, 并维护和 Nio api 之间的交互

worker 的个数在创建 conm 的时候决定, 默认配置中通过 `java.lang.Runtime#availableProcessors` 决定, 也就是 jvm 检测到的当前可用处理器数.

## 总结和体会

关于实践

1. 不应该在callback中使用耗时的操作, 这会导致阻塞io线程, 从而降低io部分的工作效率, 成为系统瓶颈.
2. 实际使用中发现，保持长链接的情况下， 存在由于链接到期断开导致刚刚放入连接池的新链接瞬间断开的情况， 这种情况需要按需重新发起链接. 我这边统计到的发生率不超过 0.1%.
