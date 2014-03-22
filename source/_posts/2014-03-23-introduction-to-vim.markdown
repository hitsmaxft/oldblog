---
layout: post
title: "Vim, 经验之谈"
date: 2014-03-23 00:21:44 +0800
comments: true
categories: [tutor, vim]
---

Vim, 老生常谈的话题了. 

> the editor

在程序员之间谈编辑器是一个敏感话题, 会有很多人跳出来挑起圣战, 还是蛮头疼的一件事情.

从Vim开始, 是一种有别于IDE的编码方式, 但仅仅是方式, 就像是拿叉还是拿筷子, 取决于你吃西餐还是中餐.

无论使用的是文本编辑器还是 IDE , 都是服务于编写源代码这么个目的. 它们之间的区别在于, 前者在编码中与环境互动, 后者则是提供一个既定的环境完成编程. 相同的是, 两者都会不断改善.

过程有所不同, 但目的是相同的. 深究下去也就是比较赛百味和麦当劳的口味罢了.

Vim 提供了两个基本的功能框架, 构成了其复杂的生态环境.

首先是模式切换. 

功能繁多的键位操作, 基于 `Normal` 和 `Insert` 两种模式的交替进行. 正是`模式`这个特性, 使得vim有别于其他以组合键方式进行操作的编辑器(or IDE). 在`Normal`模式下进行文本操作, 在`Insert`模式下进行文本输入, 并且可以通过`:map`系列命令简化和扩充可用的操作.

除了 hkjl asdcx 这些简单的移动和crud, vim 还支持按键序列的组合. 

* 删除一个单词? `de`
* 删掉10行? `d10j`
* 清空整个文件? `ggdG`

是不是有种玩街霸拳皇搓招的快感呢 XD

还有基于文本对象(text-objects^[1])的操作, 比如 `yi(` 表示 `yank inner () block`,  按下 `p` , 那么屏幕上将复制一份()里面的文本. 如果换成 `ya(`, 则表示包含`(`整块文本.

```
(text) => (text)text
[text] => [text]text
```

^[1]: 更多内容见 `:h text-objects`

此之外, 还有`command`和`visual`, 分别用于输入命令,和可视化文本选择.

比如需要一个命令的输出, 比较~~笨~~常规的方法就是去shell里面执行再重定向到文件, 然后通过vim打开进行合并. 可是 vim 支持`!`直接执行shell命令, 再加上`r`命令重定向到dang'当前文本就行了. 举个例子, 当前系统时间可以这么拿到.

```
:r!date
```

第二, 这是 `script`, 脚本支持. 

前面提到的`command`模式, 就是vim脚本的交互式shell. vim丰富的扩展功能, 也就是 plugin , 一个又一个脚本, 放置在不同的目录中, 经过不同的顺序加载实现的.

值得一提的是, vim 支持通过 python, lua 等语言的解释器, 编译的时候加上对应选项, 就可以在vim脚本中调用额外的python或者lua脚本, 通过这些语言的lib, 是的vim更加强大了.

由于vimscript相当复杂, 这里就不介绍了, 顺便说说plugin的管理和推荐.

常规的安装流程, 就是把脚本一股脑复制到 ~/.vim 目录中去, 个个插件的 autload  plugin 目录下文件会混杂在一起, 管理复杂, 想删都删不了. 

这里推荐使用, `pathogen` 和 `vundle` 进行组合管理.

pathgen和 vbundle 是两个类似的插件管理器, 区别在于 vbundle 支持配置自动去github下载脚本, 省了不少同步的麻烦. 

并非所有的vim插件都支持通过 github 下载, pathogen 就派上用场了, 虽然它只支持手动管理. 将插件所有内容解压到 `~/.vim/bundle/{plugin name}/`, pathogen 在vim启动过程中会自动加载所有的插件.

这里附上我的配置文件内容.

```vim

if version > 703

    let g:pathogen_disabled = [ 
            \,"conque_term"
            \]
    call pathogen#runtime_append_all_bundles() 

    call vundle#rc("~/.vbundle/")
    source $HOME/.vim/conf.d/vundle.vim
endif

```

通过pathogen加载vundle和其他非github插件, 再通过vundle加载其他的插件.

将vundle的 bundle 目录放置于 ~/.vim 之外, 好处是可以直接将 整个 vim 目录软链到网盘目录里, 随时保持同步, 而 vundles 则手动更新.

推荐下我自己在用vunle清单, 也就是上面的 `$HOME/.vim/conf.d/vundle.vim` 的内容.

```
" 视觉系
Bundle 'Lokaltog/vim-powerline'
Bundle 'altercation/vim-colors-solarized'

" 文件搜索 编码转换
Bundle 'kien/ctrlp.vim'
Bundle 'mbbill/fencview'
Bundle 'rking/ag.vim'

" buffer, 目录, 任务, tag浏览
Bundle 'scrooloose/nerdtree'
Bundle 'fholgado/minibufexpl.vim'
Bundle 'majutsushi/tagbar'
Bundle 'vim-scripts/TaskList.vim'

" 代码补全, 生成
Bundle 'SirVer/ultisnips'
Bundle 'Shougo/neocomplete.vim'
Bundle 'scrooloose/nerdcommenter'
Bundle 'DoxygenToolkit.vim'
Bundle 'Rip-Rip/clang_complete'

" 语法检查, 调试
Bundle 'scrooloose/syntastic'
Bundle 'joonty/vdebug'

" 其他
Bundle 'msanders/cocoa.vim'
Bundle 'gmarik/vundle'
Bundle 'L9'

```

总的来说, vim 学习成本比较高, 针对不同语言需要熟悉并定制, 才能拥有对应的功能. 
我同时也用phpstorm, 对于那些觉得不错的功能, 会考虑在vim中寻找相同的功能的plugin. 
语法检查, 补全这些有限支持也就够了, 重构功能也就是偶尔体验体验.毕竟, 这是用工具编码, 而不是工具教你如何写代码.
 
写代码的时候, 开启终端, 打开vim, 调用一下shell.
只想浏览代码的时候, 我还是回开下 phpstorm , 配置好库和环境, 通过鼠标穿梭于文件之间. 
至于两者的差别, 呵呵. 物尽其所用罢了.
