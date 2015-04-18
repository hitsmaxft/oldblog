---
layout: post
title: "nginx 路由杂谈 - 基础指令"
date: 2015-04-18 19:22:40 +0800
comments: true
categories:
- nginx
- web server
---

> 在书写 nginx 路由规则的时候， 得保证规则配置的规范， 否则维护起来成本很高。
> 本文简要地讨论了在 rewrite 模块的基础实现的简单路由规则，并解释了常用指令的使用细节。


## rewrite 与 location

首先，这是一个简单 nginx `server` 配置:

```
service {
    root /service/http;
    index index.php;

    rewrite ^/api/(.*)+ /index.php?app=api&method=$1 last;
    rewrite ^/index.php/(.*)+ /old_api_warning.html last;

    location / {
    }
    location = /index.php {
    }
}
```


上面的例子中， 指定的类型可以分为两种。

* `server`,`root`, `index`, `location` 是 http 模块提供的指令
* 而 rewrite 则是 `rewrite` 模块提供的指令之一， 同类的还有常用的 `if`

简单地概括， nginx 的处理配置流程如下

1. 进行 rewrite 规则匹配，根据命中规则改写 location。
1. 如果没有被其他指令中断并退出， 则进入 location 匹配。
1. 执行匹配成功的 location 区块中的指令。

而对于 rewrite 系列指令和 location 指令的安排， 在书写配置的时候， 应该先定义好 location ， 因为 location 是和 nginx 和后端服务沟通的桥梁。
再根据 location 的需要， 使用 rewrite 将各种各样用户输入的 url 正确地映射到 对应 locaton ， 由 location 中的 proxy 指令转发给后端。

比如一个常规的 php 站点，有以下类似的 location 配置。

```
# 静态资源
location /assets {
    root /service/http/assets/;
}
# 入口 php 脚本 位置
location /index.php {
    root /service/http/phpsrc/webroot/;
       # ... 若干 fast-cgi 代理规则
}
```

    注意， 这份配置省略了不必要的指令，能够满足大部分的 php 应用的需要。

有时候，为了优化 url 的展示，或者兼容旧应用的 url 规则， 只是这么简单的两条 location 配置自然是不够用的。这时候我们可以通过 `rewrite` 规则进行弥补。

```
rewrite /index.html$ /app/page-$1+/([\w]+); //兼容旧 url
rewrite /app/page-(.*)+/([\w]+) /index.php?app=api&page=$1&id=$2& break;
```

请注意这两条 `rewrite` 规则区别的在最后一个 `flag` 参数 `break`

     可用的 flag 参数还有 `permanent`, `redirect`, `last` 等， 后面分析它们之间的差别。

多出来的 `break` 参数作用会使得 `rewirte` 指令的跳转方式发生变化:

* 没有 flag 的 rewrite 指令完成 location 改写之后 ，继续往下寻找其他 `rewrite` 规则， 看看有没有符合要求的。如果没有， 那么进入 `location` 匹配。
* 带 `last`， 跳过其他所有 `rewrite` 规则， 进入 `location` 匹配

这里简单总结一下:

* 如果 某一条 `rewrite` 规则命中直接转发给后端应用，那么应该加上一个 `break` 标记。
* 如果 `rewrite` 规则起补充作用，还需要其他规则配合完成， 那么不带第三个的 `flag` 参数。

以上说的 `rewrite` 规则属于 nginx 的内部重定向规则， 也就是说， 用户外部看到的 url 依然是他输入的 url ， 而转给后端应用的 `$uri` ， 则已经是 nginx 改写之后的结果。

    如果需要进行显式地外部重定向， 需要借助 `redirect` , `permanent` 这两个 flag 进行 302 和 301 重定向，它的行为和 `break` 类似， 区别在于 nginx 会中断流程， 通过 http 请求向用户端返回重定向的影响，也就是这次请求不需要进过后端服务， 由 nginx 全职负责。

    `rewrite /error.html$ /error2.html redirect;`

### break 和 last 区别

这两个 flag 都会中断当前 rewrite 流程， 不再继续匹配后续的 rewrite 指令。

wiki 上是这么写的

> stops processing the current set of `ngx_http_rewrite_module` directives and starts a search for a new location matching the changed URI

如果是在 `server` 的顶级部分， 那么它们的作用是相同， 跳过剩下的 rewrite 指定， 进入 locaton 匹配。 如果 rewrite 是在 if 内部， 和直接放在 server 下级的  rewrite 行为是一致的。

```
server {
    rewrite /error1.html /error.html break;
    rewrite /error2.html /error.html last;
    if ( $arg_version = "1.1" ) {
        rewrite /error3.html /error.html break;
    }
    location = error.html {
    }
}
```

以上的 rewrite 的跳转行为是相同的， 进入 location 匹配流程。

而两者的区别， 在于当 rewrite 指定存在于 `location` 区块中时, 见例子

```
location = index.php {
    if ( $arg_q  = "" ) {
        rewrite /index.php /error.html last;
    }
}
location = error.html {
    if ( $arg_test ~= "" ) {
        rewrite /error.html /error-test.html break;
        root /service/http/asset;
    }
}

```

第一个 location , 如果 `q` 参数为空， 那么将展示错误页面。
第二个 location ， 如果 `test` 参数不为空， 通过 rewrite 规则，使用另外一个 error-test.html;

    注意， 这里并没有一个 `location = error2.html` 的规则

从这个 case 的结果， 可以明显得区分两个指令之间的细微差别

* last 跳出 location 块， 重新进行 location 匹配
* break 跳过 location 下的后续 rewrite 规则， 执行其他指令。

所以 last 的特殊在于重新进行 location 匹配，  这也就是为什么会从

`locaton = index.php`

直接转向

`locaton = error.html`

所以一般情况下， 使用 `break` 指令会相对安全， 不会造成循环重定向。

比如：

```nginx
location = error.html {
    rewrite /error.html /error.html  break;

    if ($arg_q = "") {
        return 404 "not q";
    }

    fastcgi_pass 127.0.0.1:9000;
    #其他 fastcgi 配置
}
```
```nginx
location = error.html {
    rewrite /error.html /error.html last;
    fastcgi_pass 127.0.0.1:9000;
}
```

第一个例子，完成了 rewrite 指令之后，break 指令使得后续的其他 rewrite 规则失效, 接着进行 proxy.
第二个例子会导致不断地进行 location 匹配， 最终导致 nginx 返回 500.

返回结果是这样的：

```
HTTP/1.1 500 Internal Server Error
Server: nginx/1.6.3
Date: Sat, 18 Apr 2015 12:10:49 GMT
Content-Type: text/html
Content-Length: 192
Connection: close
```

    注：nginx 会记录同一条 rewrite 规则的执行次数，如果超过10次，将自动触发 500 进行自我保护。

## return 指令的应用

在 `rewrite` 模块中， 还有一条有用的指令 `return`, 用于直接返回客户端指定的状态码。
甚至支持指定文本内容和url， 相比起使用 `rewrite` 指令302进行曲线救国，要简便地多。

```
location = /index.php {
  if ( $arg_q = "" ) {
    return 302 /page_not_found.html;
  }

  if ( $arg_id = "" ) {
    return 404 "page not found";
  }
  return 200 "hello";
}
```

对于常规的基于 url 提供服务的应用， 基础的 rewrite 指令配合已经足够完成大部分任务。
下一篇文章再聊聊基于条件，变量和更加复杂的上下文, 完成进行路由规则匹配.
