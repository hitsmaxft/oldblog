---
layout: post
title: "PHPDaemon 异步编程"
date: 2014-03-24 03:39:27 +0800
comments: true
categories: 
- async
- php daemon
- php
- network
---

phpdaemon, 基于异步网络请求的应用服务框架

一直以来, php给人的印象就是一个单进程同步的编程语言, 其魅力在于简单, 直观,
学习成本低, 容易上手, 所以市场占有率很高.

流行的php模式, 从几年前的 apache + mod-php, 到今天的nginx + php-fpm,
在php编程层面上都是一直的.
开发者的开发的代码都是居于”php进程会同步执行所有的代码逻辑之后退出并释放”
这么一种假设, 不用考虑多线程的锁问题, 大多数情况下也不用考虑内存泄露.
虽然实际情况下, php进程受上层管理界面的影响, 并不是像 cli
模式那样执行一次并退出, 但对开发者而言, 其中细节可以忽略.

回到 phpdaemon 本身吧,
phpdaemon是一个在event和eio两个pecl扩展之上开发的异步网络框架, 类似于 python
中的 tornado , 它采用的单进程单线程异步IO的网络模型,
可用于开发异步非阻塞的应用.

前面说 phpdaemon 是单进程单线程的网络模式, 严’意义上讲是指它的worker. phpdaemon
以 master + workers 方式运行, master只负责分发任务, 在worker
进程才进行请求内容的处理工作.

在 phpdaemon 中, 一个进程中会启用多个服务实例,  每个实例对应一个 appinstance.
开发者将不同的处理逻辑封装成为不同的 app, 在异步调度中各司其职.

master 在接受到请求之后, 向 AppResolver 发起查询, Resolve 根据上下文,
返回一个对应的应用实例名称. 比如需要返回一个静态文件, 那么可以返回一个
FileReader 实例, 实例的Request 方法中完成异步并向客户端输出文件内容.
而最常见的任务是返回一个动态页面的内容, 这种类型的任务, 在 phpdaemon 中抽象为
HttpRequest 类型.   一个常规cms中的前端和后台界面,
可以分装成两个不同的appInstance, 在resolver实现对应的判断逻辑,
正确路由到对应的appInstance上, 然后再决定发起哪一个 Request 类型.

这种关系有点像 MVC 框架中的C. 大部分php web 框架的 controller
中会包含多个action, 在 phpdaemon 中, `PHPDaemon\Core\AppInstance`
可以认为是controller, 而action则对应着 client\Request.

为什么这里要分成 AppInstance 和 Request 两种类型呢?
这里涉及到并发环境中的内存管理问题. 常规的php应用是同步的, 从来不考虑这个问题.
一个实例最终会被释放, 多个请求之间并没有共享关系. 但是在phpdaemon中,
并不是这么回事. worker进程一段时间内一直驻留内存, 需要管理每次请求中用到的资源. 

服务器的配置是不变的, 可以常驻内存. 对下层接口的调用结果, 不同任务之间是独立的,
所以任务完毕后应该释放掉.   

所以, 在并发环境下, AppInstance 不断生成新的 Request
对象处理任务并存储在队列当中, 而 Request 在进行网络请求的时,
会交出cpu并进入休眠, 等待io事件将自己唤醒. 当 Request 完成任务并返回结果,
AppInstance 将结果返回, 并将该 Request 实例从内存中释放.

上文提到的 Request , 并不是一般认为的 网络请求, 而是对应负责处理网络请求的
Service. 那么应用的网络请求又是如何发起的呢?

这里先解释下 phpdaemon 中的 pool 和 connection.

Connection, 顾名思义, 就是php代码中发起的异步网络请求, 比如常见的 curl.
在异步请求中, 应用发起很多网络请求, 通过事件进行调度.
所以同一时间存在多个网络请求, 只不过有的是活跃状态, 有的挂起等待后端返回.
存储和管理这些 Connection 的负责人, 就是 Pool. 这就是常说的连接池.

Java语言中的连接池, 一般通过多线程实现的. 而 PhpDaemon 中的连接池,
则是在libevent上实现的单进程单线程异步连接池.
