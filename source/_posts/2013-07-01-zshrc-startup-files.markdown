---
layout: post
title: "Zshrc: Startup Files"
date: 2013-07-01 22:54
comments: true
categories: shell environment zshrc
---

shell 环境有必要保持一定的整洁性, 毕竟多台机器之间多少要做些配置环境上的同步.
本文简单讨论了 zsh 使用中配置文件规范性.



zsh 的 man 手册中提到, zsh 的配置加载顺序如下

最高优先级, 所有情况下都会加载

```
/etc/zshenv
$HOME/.zshenv
```

login shell 配置, 两组文件的区别在于, 中间存在着 zshrc 的加载

```
/etc/zprofile
$HOME/.zprofile

/etc/zlogin
$HOME/.zlogin
```


交互shell配置

```
/etc/zshrc
$HOME/.zshrc
```

三种配置文件的差异, 在于加载的时机不同.

`/etc` 目录下的配置文件会影响机器上的所有用户, 相比起 $HOME 配置文件, 加载顺序优先级比较高
但是除了系统管理员, 大部分情况下就一个人用的机器, 没多大关心的必要.
 平时除了偶尔切换到 root 下, 基本不会考虑多用户的情况. 这里就不过多涉及了.

**env配置**


顾名思义属于环境配置, 只要新启动的shell实例, 都会重新加载. 这就意味这个文件的内容会重复加载, 不适合加入需要依赖io等耗时的操作, 影响了shell的执行速度

我通过这个配置文件, 区分不同机器, 比如

```
VIMID="hits-office"
```

方便 vim 中可以根据`$VIM`加载工作环境下的特殊配置, 比如以 gbk 作为默认文件编码 :( .

**login shell**

部分发行版在这个配置文件中加入了`. $HOME/.zshrc` , 这种做法也意味着, 两者比较难区分.

**zshrc**

鉴于大部分人都是在交互式环境下使用 shell , 也就意味着这个文件是最常修改的配置文件, 比如著名的`oh-my-zsh`, 其入口文件便是`$HOME/.zshrc`. 

刚接触shell的人, 对于 login shell 和 interactive shell 常常傻傻分不清.
由于对与shell的定制和配置往往是增加 function, alias, 以及补全等等, 都是为了方便shell操作, 所以大多数情况下都是修改`zshrc`实现的.

那么 `.zprofile` 是不是有点多余呢? 以下讨论 `.zprofile` 可以发挥作用的场景

场景#1: `export A=abc` 定义全局常量

仅仅在登入系统时执行一次, 某些需要调用`hostname`之类耗时的操作也可以放在这里, 不会因为重复执行而浪费 CPU 运算.

场景#2: 初始化虚拟环境

在安装 `rvm` 或者 `pythonbrew` 的情况, 需要在当前环境加载相对应的初始化脚本, 才能使用 `rvm use xxx`, `pythonbrew use xxx` 等方式修改当前 interceptor.

{% codeblock ~/.zprofile %}

#load default profile config
. $HOME/.profile

# Load RVM into a shell session *as a function*
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
# Load pythonbrew into a shell session *as a function*
[[ -s "$HOME/.pythonbrew/etc/bashrc" ]] && source "$HOME/.pythonbrew/etc/bashrc"

{% endcodeblock %}
