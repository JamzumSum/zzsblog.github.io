---
date: 2021-08-18
author: JamzumSum
html_meta:
    keywords: crawler, tcaptcha, Qzone, Qzone2TG
---

# TCaptcha的检测机制

## 前言

由于我最近在做[QQQR][QQQR]的验证码部分, 比如![这种](_static/tdx/tcapcha-screenshot.png), 在Qzone2TG的前面一些版本里, 我都是用selenium爬取图像配合模拟拖动来做的, 当然也需要一点点的cv技术.

[QQQR]: https://github.com/JamzumSum/QQQR "a simulation of tencent login HTTP protocol"

但上面这种办法虽说理论可行, 但并非真的万无一失. 事实上, 在一开始开发的时候跑通之后, 这个办法就再也没好用过. 等到后来我用selenium抓二维码登录, 乃至[QQQR][QQQR]的第一个版本能够实现协议二维码登录的时候, 我就更不想用这种不可靠的办法了.

直到[QQQR][QQQR]的开发到了`2.3.0`, 我囿于一种执着重新开始研究验证码的破解之道, 在长达十余天乃至可想见的数十天中仔细研究抓包结果和TCaptcha相关的js代码, 我才终于意识到, 使用selenium这些模拟登录的办法究竟处在一种怎样危险的境地.

截至目前, 我还没有搞定`23003 网络环境异常`的问题. 我相信唯一让我露馅的可能就在于验证码检测的时候. 尽管如此, 我仍然发现了很多tx检测浏览器特征的"小动作".

## tdx.js

在加载TCapcha的时候, 页面会载入`tdx.js`. 这整个一个JS就是一个...虚拟机, 用我自己能明白的话来说就是一个指令集+栈+纸带(雾)的组合. 我在这上是个外行, 有的时候不明白加密何至于搞这么大的阵仗. 但不得不承认, js混淆+这种叫做`TENCENT_CHAOS_VM`的技术的确可以大大阻碍我们这些不守规矩的人(嚣张. 但可惜, tx自己搞了个什么大赛, 这种虚拟机的机制已经参赛的大佬们看透了. 我这种一开始蒙在鼓里的人, 在Github上一搜, 也就明白得差不多了.

上面所说的, 指令集+栈+纸带的组合其实就完全确定了程序的逻辑. 我的目的也就是在这一团糊糊里找出tx究竟搜集了什么信息进而判断出我不对劲的 :(

我首先尝试了一些手动反汇编的办法, 给每个命令加输出, 类似编译器. 但, 应该说是受限于我本身的水平, 得到的结果不够准确, 没法交给机器处理, 也就没大用. 不过从另一个角度, 这一步让我真正地深入到了指令运行的内部, 从而为后面的调试打下了基础.

## 到底检测了什么

背景和步骤到此为止, 没有再多说的必要了. 下面就是列出一些检测的条目, 给包括我在内的初出茅庐的人们一点警醒: selenium等等这些模拟手段没那么"真".

### 自动化测试工具

这一类比较泛用, 是确切的用来检测客户端是否受自动化控制的办法.

- window.callPhantom
- window._phantom
- window.WebPage
- window.fxdriver_id
- window.__fxdriver_unwrapped
- window.domAutomation
- window.ubot
- window.CasperError
- window.casper
- window.patchRequire
- navigator.webdriver
- document.$cdc_asdjflasutopfhvcZLmcfl_
- document.__webdriver_script_fn

这些属性, 正常应该为`undefined`. 在使用某些特定工具时, 这些值会被设置, 从而可以直接断定当前访问是受自动化控制的.

比如`window.navigator.webdriver`, 使用selenium时该项会设为`true`;
比如`window.document.$cdc_asdjflasutopfhvcZLmcfl_`是chromedriver的一项特征, 正常的浏览器并没有这个键.

### 环境信息

`tdx.js`也读取了这些信息, 分析它们可以判断当前客户端处在一个什么样的环境. 虽然没有证据. 但我相信`23003`的错误应该就在此检出(雾

#### document.mozHidden

似乎是检查标签页是否可见, 在chromium上为`undefined`

#### navigator

- navigator.userAgent
- navigator.appVersion
- navigator.platform
- navigator.cookieEnabled
- navigator.languages
- navigator.vendor
- navigator.appName
- navigator.plugins
- navigator.getBattery

#### document & location

- document.charset
- document.cookie
- document.referrer
- document.documentElement.clientWidth
- document.documentElement.clientHeight
- document.getElementById
- location.href

#### screen & other display properties

- screen.colorDepth
- screen.width
- screen.height
- screen.availHeight
- screen.pixelDepth

- window.innerWidth
- window.innerHeight

上述这些属性和方法均是被检查的对象. 这些项恐怕也是selenium等工具的优势所在: 它们能和正常浏览器保持一致.

当然, 对于普通的爬虫来说, 恐怕不会有这么细致的检查, 毕竟通常我们只爬HTML, 这些情形基本是遇不到的.
但毕竟[QQQR][QQQR]是要直接和验证码作斗争, 其本质不仅仅是爬虫, 使用的工具也是`requests`+`nodejs`的基础组合, 因此这些都是要注意的对象.

### 用户操作

人类访问是需要操作的, 而爬虫没这个必要. 因此`tdx.js`也收集用户的动作.

- 客户端类型(手机/PC)
- 点击等(似乎并不编码于`collect`字段, 而是`action`字段)
- 坐标系(验证码位置, 缩放比), 以及拖动拼图时鼠标的时间-坐标关系
- ...(是否有其他的还不清楚)

> 未完待续
