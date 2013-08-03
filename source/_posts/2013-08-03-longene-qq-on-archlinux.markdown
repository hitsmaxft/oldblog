---
layout: post
title: "Archlinux Longene QQ 轻松上手"
date: 2013-08-03 01:04
comments: true
categories: linux-desktop, app, qq, archlinux
---
### 通过 Aur 安装相关包

完整安装 longnen qq 需要以下几个包

* `wineqq` 主要包, 包含完整的 longene 环境和 qq 2012
* `opendesktop-fonts` 提供 odokai 字体, 用于中文字体显示, 不装可能导致方块字
* `ttf-ms-fonts` 提供 arial 用于中文显示

装上了就能跑了, 基本功能都没问题.

### 可能遇到的问题

**英文半角符号显示错位**

    字体问题, 未解决

**没有声音**

需要安装声音相关的包, 64位系统装上第一个, 32位选择第二个.

> * lib32-alsa-plugins 
> * alsa-plugins 

64 位系统需要启用 multi 仓库才能找到 lib32 相关包, 具体操作见 [wiki:multilib](https://wiki.archlinux.org/index.php/Multilib) 

简单说, 就是找到 `/etc/pacman.conf`, 找到以下文本, 取消注释(如果没有找到就手动加入)

    [multilib]
    Include = /etc/pacman.d/mirrorlist

**fcitx无法输入**

找到 `/opt/longene/qq2012/qq2012.sh`, 将其中的

    export LANG=zh_CN.UTF-8

改为和当前系统一致的编码 (用 locale 命令查看), 比如我的是

    export LANG=en_US.UTF-8
