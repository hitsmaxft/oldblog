---
layout: post
title: "Composer 实战"
date: 2013-08-04 00:09
comments: false
categories: php, composer
---
**Composer 是什么?**

按官方文档

> Composer 是一个 PHP 的依赖管理工具. 它可以自动安装项目中声明依赖的库.

**为什么要使用 composer , 而不是传统的 pear 等?**

Compoesr 是一个类似于 ruby-rubygems, nodejs-npm, python-pip 的依赖管理工具. 相比起pear简单的安装功能, Composer 注重包之间的依赖关系, 每个项目可以独立管理自己的依赖, 而不是一股脑全部安装上. 另外也解决开源组件混乱的 autloader.

当前的 php 框架社区中活跃成员(比如 Symfony, Slim, Laravel, 以及开发中的 Yii2) 纷纷支持 composer , 旺盛的生命力已经不是 pear 之类的老掉牙的社区方案可以比拟了.

相对于 ruby, python, nodejs, php总算有了一个相对完善并且流行的包管理器.
目前社区和开发者之间的距离还比较大. pear, pyrus, composer 会继续并存下去, 给 php 开发者带来更多的麻烦.
长期而言, 打个你死我活, 最后只剩一个是符合开源社区的历史发展规律的 :)

## 简单上手

安装具体步骤参考 [http://getcomposer.org](http://getcomposer.org) , 简单运行一下命令

    curl -sS https://getcomposer.org/installer | php
    
最新版本的 composer.phar 就下载到本地了.

相比起 `npm` `pip` 等被大部分发行版内置或者仓库支持的包管理器, 这种自己手动下载方式相对不够友好, 毕竟 composer 的知名度暂时还追不上 pear 等老牌工具, 被周边社区还需要一定的时间.

常用命令官方站点上有简(jian)洁(lou)的文档, 

    install
    search
    show
    update
    create-project
    
按前面的方式下载之后, 使用composer 需要用 `php composer.phar help` 这样丑陋的方式, 嫌麻烦可以手工移动到phar到 `$PATH` 对应的相关文件夹中去, 顺便改名为 composer 会顺眼一些.

> 注意文件的可执行权限, (chmod +x composer)

用法是比较简单的, 关键是发布者和用户注意用 composer.json 准确地描述相关的依赖和加载细节.

## 依赖入门

先给一份简单的例子

* 一个简单的工具集, 版本号 1.0.0
* 依赖 net/http , 用来处理http请求, 而不是直接用 Curl
* 开发模式下依赖 phpunit, 方便运行单元测试
* 使用 psr-0 规范的加载方式, 源代码位于 src 目录中, 将 `Utils` 映射到该目录

{% codeblock composer.json lang:json %}
{
    "name" : "example/utils",
        "description" : "utils",
        "version" : "1.0.0",
        "type" : "library",
        "license" : "proprietary",
        "authors" : [
        {
            "name" : "Mr Right",
            "email" :  "mr.right@taobao.com"
        }
    ],
        "autoload" : {
            "psr-0" : {
                "Utils\\" : "src/"
            }
        },
        "require" : {
            "net/http" : "*"
        },
        "require-dev" : {
            "phpunit/phpunit" : "*"
        }
}
{% endcodeblock %}

例子包含了一个包信息中常见字段, `require` 不一定需要, 但这是依赖管理的精华所在, 后续章节再仔细说明.

> 刚开始可能对 composer.json 的语法不太熟悉, 可以通过 `composer validate` 检查自己书写风格是否符合规范

## 文件目录结构

还是例子先行

    trunk/
        src/
            Utils/
                NetWork/
                    WangWangMsg.php
        
        composer.json
        
        phpunit.xml
        
        tests/
            bootstrap.php
            Utils/
                Network/
                    WangWangMsgTest.php
           

composer 提供了灵活 `autoload` 机制, 文件的摆放方式可以很灵活, 但过分奔放的 style 不是好事. 以上结构参考自 `Monolog`, 一个强大的日志工具, 也考虑到了phpunit的使用.

> 推荐将 phpunit 也作为依赖之一加入 `require-dev` , 方便开发调试. 与 `require` 字段不同的是, 只有显式使用 `composer install --dev` 才会安装上 `require-dev` 的相关组件.

对于用户而言, 使用第三方包, 最不希望的应该就是牵扯进 autoloader 的细节. 以往的 php 源码包都有自带autoloader 的毛病, 复杂的 autoloader 交织在一块往往给使用者造成麻烦. 相比之下 composer 使用统一的方案, 优雅地解决了这个难题. 

### autoloading 与 命名空间

composer 目前支持三种 autoloading 方式

- `psr-0` 根据 psr-0 规范命名空间和文件路径进行类加载, 最佳实践方式.
- `classmap` 生成类与文件的映射关系文件, 安装时自动扫描并生成.    
- `file` 可以理解为预先加载指定文件, 每次请求中都会执行

这里简单说说 psr-0, 参考 utils 这个包的配置文件

{% codeblock composer.json lang:json %}
    "autoload" : {
        "psr-0" : {
            "Utils\\" : "src/"
        }
    },
{% endcodeblock %}

在安装之后 composer 生的classmap 信息文件之一, `vendor/composer/autoload_namespaces.php` 中可以看到以下代码

{% codeblock lang:php %}
<?php
    return array(
        'Utils\\' => array($baseDir . '/src'), // HERE!
        'Symfony\\Component\\Yaml\\' => array($vendorDir . '/symfony/yaml'),
        'Net_' => array($vendorDir . '/net/http/src'),
    );
{% endcodeblock %}

可见进行相关类的自动加载是, composer autoloader 通过查表和路径搜索的方式, 正确地找到类相关的代码.

比如一下代码

{% codeblock lang:php %}
<?php
    new Utils\Network\WangWangMsg();
    //or new Utils\Network_WangWangMsg();
{% endcodeblock %}

实际运行时, 实际运行的逻辑类似于以下的伪代码

{% codeblock test.php lang:php %}
<?php
    //composer
    $baseDir = $root . 'vendor/example/utils';
    include $baseDir . '/src/Utils/Network/WangWangMsg.php';

    new Utils\Network\WangWangMsg();
{% endcodeblock %}

## 项目中使用 composer

在 composer 设计理念中, 项目和包并不是两种割裂的定义, 所有的源代码都归属于对应的包, 包与包存在依赖关系.
虽然在实际业务开发中, 暂时没哪个项目如此纯粹, 但这并不影响 composer 的应用.

以下是一个简单的搜索结果页应用如何书写 composer.json.

{% codeblock composer.json lang:json %}
{
    "repositories" : [
        {
        "packagist":false //禁用外部仓库
        },
        {
            // 引入内部代码仓库(svn)
            "type" : "svn",
            "url" : "http://.../mido",
            "branches-path" : false,
            "tags-path" : false
        }
    ],
    "require"  : {
        "mido/mido":"dev-trunk" //使用 svn 中的 trunk 分支
    }
}
{% endcodeblock %}

运行 `composer install` 时, composer 预先扫描所有的 repositories, 将 require 字段中的包更新载入到 vendor 目录中去, 并生成相关缓存信息和自动加载代码.

然后呢?

在应用 bootstrap (对于大部分项目, index.php 可能是一个更加容易理解的名字)代码中加入以下代码

{% codeblock index.php lang:php %}
<?php
    require /path/to/root/vendor/autoload.php
{% endcodeblock %}

接着项目代码中尽情使用吧, 毕竟自动加载类的细节都已经交给 autoloader 了.

## 一个例子

这里介绍怎么使用 composer 作为 yii 1.x 的依赖管理工具. (注意, yii1.x 不符合 psr-0, 直接使用 composer 比较困难)

安装好了依赖, 接下来讲解怎么处理利用代码. 这里先用yii自带的demo helloworld 作为例子.

复制文件(yii/ 目录下存放最新的yii 1.x 代码):

    cp -r yii/demos/helloworld ./
    cp -r yii/framework helloworld/protected/

目录结构如下:

    helloworld/
        index.php
        protected/
            composer.json //composer 配置信息
            framework/  //yii 框架文件
            vendor/     //composer安装依赖的位置

{% codeblock composer.json lang:json %}
{
    "repositories"  : [
        {
            "type" : "composer".
            "url" : "http://packages.phundament.com"
        }
    ],
    
    "require" : {
        "php" : ">=5.3.12",
        "yiiext/migrate-command" : "0.7.2",
        "psr/log" : "1.0.0",
    },

    "autoload" : {
        "classmap" : [ "framework/YiiBase.php", "framework/yii.php" ]
    },

    "config" : {
        "vendor-dir" : "vendor"
    }
}
{% endcodeblock %}

index.php 使用以下代码替代原来的 yii.php

{% codeblock composer.json lang:php %}
<?php
    require "protected/vendor/autoload.php";
{% endcodeblock %}


将 framework 和 composer.json 放在 protected 文件中, 方便通过 webserver 访问规则屏蔽代码文件. 文件结构可以根据实际需求做调整, 这对于 composer 来说都这可以正确找到代码.

接下来 进入 protected 目录中 运行 composer install -vvv , 更新composer信息, 生成 classmap.

    php -S 127.0.0.1:8080
            
应用就跑起来了.

## 主搜索应用前端公共库 mido 改造实战

mido 是我们使用一个公共包, 负责了大部分引擎请求的细节, 在业务代码中屏蔽了请求, 解析等细节.

**1. 规范svn路径**

严格区分`trunk`, `tag`, `branches`. composer 对于svn的支持相对弱一些, branches 和 tag 都可以别名, 或者禁用, 但是 trunk 好像必须要有.

目前 mido 只有 trunk 一个目录, 并没有严格按版本号发布, 所以项目的 `require` 字段填上 `"mido/mido" : "dev-trunk"` 就行了.

    /trunk
    /branches/2.0.1-beta
    /tag/1.0.0

**2. 源代码命名空间处理1 : 用 \\ 替换 \_**

{% codeblock lang:php %}
<?php
     Mido_Exception -> Mido\Exception
{% endcodeblock %}

严格意义上说, 这并不是必须的. composer 支持 `A_B_C` 这种常规的写法, 在配置中写上 `"Mido_":""` 也行

**3. 源代码命名空间处理2 : 命名空间导入**

*ps: 需要特别注意全局命名空间里的类名,包括不支持命名空间的库和扩展, 如ArrayAccess, Memcache*

{% codeblock lang:php %}
<?php
    namespace Mido\Engine;
    use Mido\ComponentBase;
    
    class Kingso extend Base {}

    namespace Mido\Engine\Kingso;
    use Mido\ComponentBase;
    use Mido\Engine\RequestBase;

    class Request extend RequestBase implements \ArrayAccess {}
{% endcodeblock %}
     
当然, 以根命名空间的方式书写代码也是可以的, 看团队如何制定代码规范.

**3. 源代码命名空间处理: 保留关键词**

前缀式命名类没有和保留关键字冲突的问题, 但改用命名空间时, 在当前域需要注意扩展和内置提供的全局class, 虽然丑了点, 但是结果是好...

mido 中内置了部分常用的 helper, 比如以下的 'a.b.c' 的多维数组访问 helper. 原来使用了 Array 这个关键字(呃, 因为 php 不关心大小写)

{% codeblock lang:php %}
<?php
//old style
Mido_Helper_Array::get($items, 'items.title');
//new style
Mido\Help\MArray::get($items, 'items.title');

{% endcodeblock %}

**4. 组件别名配置更新格式**

为了满足多个相同引擎单例组件的使用, 配置文件里定义了映射关系, 偷懒直接写上根命名空间.

{% codeblock lang:php %}
<?php
//修改前
"kingso" => "Mido_Engine_Kingso",
//修改后
"kingso" => "\\Mido\\Engine\\Kingso",

{% endcodeblock %}

嗯, 全部的迁移成本如上.

## 总结

**好处**

- 方便资源共享, 方便代码分发
- 项目源代码结构清晰 
- 自带autoloading机制, 简化加载流程
- 社区类库支持强大
- 与国际接轨, 高端大气上档次 :)

**使用成本**

- 独立执行文件, 暂时没有比较好的工具集成, 暂时没见那个发行版内置.
- 对于频繁改动, 代码耦合严重的应用, 并没有明显成效.
- 需要遵守目录结果规范
- 支持还不够广泛, 许多第三包需要定制安装

## 附

* [troubleshooting.md](http://getcomposer.org/doc/articles/troubleshooting.md)
* [Composer: Part1 - What & Why ](http://nelm.io/blog/2011/12/composer-part-1-what-why)

*vim:ft=markdown*
