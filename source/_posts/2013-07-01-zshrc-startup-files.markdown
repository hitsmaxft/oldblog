---
layout: post
title: "Zshrc: Startup Files"
date: 2013-07-01 22:54
comments: true
categories: shell environment
---

shell 环境有必要保持一定的整洁性, 毕竟多台机器之间多少要做些配置环境上的同步.

## 系统级别

/etc 目录下的配置文件会影响机器上的所有用户, 相比起 $HOME 下配置文件, 加载顺序优先级比较高

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

首先这里先排除 /etc 下的配置文件, 除非是系统管理员, 否则像我这种单机用户, 就一个人用的机器, 没多大关心的必要. 平时除了偶尔切换到 root 下, 基本不会考虑多用户的情况. 这里就不过多涉及了.

*env配置*

顾名思义属于环境配置, 只要新启动的shell实例, 都会重新加载. 这就意味这个文件的内容会重复加载, 适合放置常量, 比如 `EDITOR=vim`

我通过这个配置文件, 区分不同机器, 比如

```
VIMID="hits-office"
```

方便 vim 中可以根据`$VIM`加载工作环境下的特殊配置, 比如以 gbk 作为默认文件编码 :( .

这份配置千万不要加入什么 echo 或者其它涉及阻塞操作的调用, 说多了都是泪.

*login shell*

部分发行版在这个配置, 会读取 zshrc , 这种做法也意味着, 两者比较难区分.

*zshrc*

鉴于大部分人都是在交互式环境下使用 shell , 也就意味着这个文件是最常修改的配置文件, 比如著名的`oh-my-zsh`, 其入口文件便是`$HOME/.zshrc`. 

刚接触shell, login shell 和 interactive shell 常常傻傻分不清. 由于对与shell的定制和配置往往是增加 function, alias, 以及补全等等, 都是为shell操作服务的, 所以适合放置于 bashrc.

那么 profile 是不是有点多余呢? 一下讨论 profile 可以派上用场的场景

`export A=abc` 这种定义全局常量的做法, 前面说的适合放在env中, 但是实际上并不是每次重新打开shell都需要重复这么一个操作, 建议将这部分配置移动至 zprofile. 这样子仅仅在登入系统时产生影响, 某些需要调用`hostname`之类耗时的操作, 也不会因为重复执行而影响执行速度.

在安装 `rvm` 或者 `pythonbrew` 的情况, 需要在当前环境加载相对应的初始化环境, 才能使用 `rvm use xxx`, `pythonbrew use xxx` 等方式修改当前环境中的 interceptor.

{% codeblock ~/.bashrc %}

# Load RVM into a shell session *as a function*
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
# Load pythonbrew into a shell session *as a function*
[[ -s "$HOME/.pythonbrew/etc/bashrc" ]] && source "$HOME/.pythonbrew/etc/bashrc"

. $HOME/.bashrc

{% endcodeblock %}


shell 和 screen 提供 -l 使得当前shell以 login shell的方式执行, 这时候有机会重新加载 zprofile 和 zlogin. 


