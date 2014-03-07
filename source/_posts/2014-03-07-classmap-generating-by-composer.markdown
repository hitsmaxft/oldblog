---
layout: post
title: "基于 composer 的classmap生成器"
date: 2014-03-07 10:18
comments: true
categories: php composer classmap
---

近期花了不少功夫在优化线上代码构建方案上, 过程中的一些心得会不定期更新.

先前的做法是发布之后调用一个yii的扩展, 生成Classmap文件, 包含了类名和绝对路径的映射关系.

这么做的好处自然是在 `autoload` 上最大限度地减轻消耗, 每个类加载 `autoloader` 仅仅需要检查一下 `classmap` , 剩下的工作交给 `apc` 就行了.

最近在进行 `composer` 迁移的过程中, 发现composer其实自带了这么一个功能: `dump-autoload`, 而且生成的 ClassMap 文件在运行时通过路径拼接, 而不是用相对路径, 同样不会影响到  `apc` 的缓存功能.

实现这个功能仅仅需要以下改动

composer.json

    {
          "autoload" : {
                  "classmap" : [
                              "src/components",
                              "src/controller"
                   ]
          }
    }


spec文件中加入

    BuildRequires: t-search-composer-cli
    /opt/tsearch/bin/composer dump-autoload

在应用初始化过程加入类似以下步骤

    $classMap = require "vendor/composer/autoload_classmap.php";
    Yii::$classMap  = array_merge( Yii::$classMap, $classMap);

对于已经用上composer的项目管理依赖的情况下, 这种做法自然不是需要的, composer 的 `install` 指定会自动完成 `classmap` 生成工作.

但是如果不需要依赖 `composer` 进行依赖管理的情况下, 通过 `dump-autoload` 指令来提供可用的 classmap, 简单而又效果显著.
