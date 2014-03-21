---
layout: post
title: "自定义 zsh 自动补全功能初探"
date: 2014-03-17 02:43:53 +0800
comments: true
categories: [zsh, script, tutorial]
---

zsh 已经不是什么新鲜事了, `oh-my-zsh` 相信很多人都已经在用了.

但`zsh` 存在一个明显的问题, 它并不是什么开箱即用的工具, 需要大量配置, 也很难上手扩充新功能, 这点和vim非常相似.

常用的 alias 和 shell script, 虽然能用上些稍微方便的特殊语法, 大部分仍和 bash 相近. 这时候倒不如使用bash的语法, 保证代码的兼容性和可移植性.

本文简单介绍如何编写zsh的补全插件, 以 Mac OS 的 launchctl 为例.

    `launchctl` 是 Mac 用于管理系统运行, 类似于 linux 的 systemd, 用于管理 LaunchAgent 加载, 是 `launchd` 的前端.

常用的 subcommand 有 unload, load, stop ,start 等等. 这里就只考虑 load 和 unload 补全功能扩充.

## 怎么开发脚本

按作者的话说, 看文档是很难学会的. 因为很少涉及脚本编写本身, 仅罗列了所有的接口.

## 基本内容

首先将脚本命名为 `_launchctl` , 放到任意目录下, 让将目录加入 fpath

    oh-my-zsh 用户保存在 ~/.oh-my-zsh/plugins/launchctl/_launchctl.
    并在 zshrc 中声明 plugins=( launchctl )
    剩下的事情就交给 oh-my-zsh

脚本头部加上以下内容

```bash
#compdef launchctl
```

第一行标记这个文件包含一个自动加载的函数. 
本质上这是一个没有形如 `function () {}` 包裹着的函数体,
和可以直接通过 `source` 导入的配置文件和脚本是不同的.

## subcommand 补全
    
subcommand 应该是子命令或者副命令的意思， 这里保留原文。

    ps: 对于 gnu 风格的命令行工具, 很少有 subcommand , 这一章节并不是必要的.

```
local -a _1st_arguments

_1st_arguments=(
    "load:Load configuration files and/or directories"
    "unload:Unload configuration files and/or directories"
    "start:Start specified job"
    "stop:Stop specified job"
    "help:This help output"
)

local label_subcommand="launchctl subcommand"

if (( CURRENT == 2 )); then
    _describe -t commands "$label_subcommand" _1st_arguments
    return 0
fi
```

这段代码还是比较好懂的, 补全所有的`subcommand`并给出提示信息.

注意 `CURRENT` 这个变量, 它标记处于命令行的第几个单词

    关于 CURRENT , 见 `man zshcompwid`

在按下tab之前, 用户输入的文本为 `launchctl<空格><tab>`, 第一个单词是主命令`launchctl`, 显然在这段脚本中, `CURRENT` 是一个大于2的整数.

对于还存在更下一级 subcommand 的补全, 以此类推.

## 文件补全

`launchctl unload` 用于关闭一个正在运行的服务, 比如nginx, php-fpm等等, 显然这里需要补全的是一个文件路径, 对应一个位于 `~/Library/LaunchAgents/` 下的 `plist` 文件.

首先假设已经实现了对应的工具函数 `_get_loaded_user_plists` , 这个函数将生成目前正在运行的plist文件列表

以下是关键代码

```bash

local expl

case "$words[2]" in 
    unload)
        local loaded_user_plists
        loaded_user_plists="$(_get_loaded_user_plists)"
        _wanted loaded_user_plists expl 'running jobs' compadd -a loaded_user_plists
        ;;
esac

return 1
```

可见对于简单参数的补全实现是很轻松的

zsh 如果发现补全函数返回0, 会将输出作为补全内容作为候补内容输出到终端中去.

因此我们只需要确定 subcommand 是我们所预期的`unload` , 将文件列表发送给用户就行了.

这里用`case` 根据 subcommand 的不同, 选择对应的补全形式.

首先注意 `$words` 这个变量, 通过打印该变量可以发现, 它是一个将用户已经输出的命令拆分的数组

比如输入 `launchctl unload` , 那么对应的words将是 `( 'launchctl' 'unload' '' )`, 最后一位是一个空字符串.

    严格地说, 这里拆分依据并不是空格
    如果你明白 $@ 和 $# 区别, 你就懂我啥意思了. 
    这里空间不够, 我就不写了 :)

## 调试脚本

开发过程中难免写错, 需要重复调试, 这里有两个小技巧.

1. echo

echo 当然是喜闻乐见的 debug 指令之一了.

2. set -x 

`set -x` 将开启zsh的 XTRACE 选项, 所有运行脚本背后的指令和参数都直接打印到屏幕, 俗称上帝视角.
如果发现不能正常执行, 那么只要运行 `set -x`, 手动输入命令后按tab. 

> ps1: set +x 恢复正常模式.
> ps2: 千万不要手残提前触发其他zsh的特性, 不然面对满屏的文本里就没啥心情继续下去了. 这里需要老老实实输入完整的命令, 在需要触发对应补全过程的地方按下tab触发补全.

由于调试信息实在太多了, 运行几次之后可以把调试窗口关了, 或者将终端的buffer清空, 总而言之, 减少输出的log以快速定位问题.

    至于更细节的内容以及相关api文档, 请看手册 `man zshcompsys`
    zsh 的~~破手册~~跟天书一样的文档也没打算解释清楚, 我能说的只有这么多了
    更多技术细节请自行深挖.

Job done!
