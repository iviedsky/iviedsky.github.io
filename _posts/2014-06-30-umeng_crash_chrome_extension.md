---
layout: post
title: Chrome扩展应用——解析友盟崩溃日志 
category : 代码 
tagline: "Supporting tagline"
tags : [Chrome扩展应用, Python, Tornado]
---
{% include JB/setup %}
---

##Chrome扩展应用——解析友盟崩溃日志[编辑中...] 

------
###一、背景：

当iOS应用程序接入友盟统计后，我们可以在友盟网站"错误分析->错误详情"页面看到应用程序的Crash Log。

然而，在该页面中看到的日志，是未经过Debug Symbol(dSYM)文件中符号表转化的,可读性并不高。

所以，博主编写了一个Chrome扩展应用，用于在页面中将Crash Log直接进行翻译。
<!--break-->
翻译前，页面内容如下：

![翻译前](/image/umeng_extension_02.png =800x1200)

翻译后，页面内容如下：

![翻译后](/image/umeng_extension_03.png =800x1200)

------
###二、解决方案：

1、打开友盟统计错误详情页面以后，点击Chrome扩展应用。Chrome扩展应用读取页面中Crash Log相关的信息，并发送至服务端。

2、服务端根据保存的dSYM文件和app文件，翻译Crash日志，并将翻译后的内容返回。

3、Chrome扩展程序将服务端返回的内容写回到页面中。

下图中，实线表示客户端请求，虚线表示服务端响应

![结构图](/image/umeng_extension_01.png =400x600)

------
###三、服务端：

技术方案：语言 - Python，HTTP服务器和框架 - Tornado


------
###四、Web客户端：

技术方案：语言 - JavaScript，平台 - Chrome


------
###五、总结

<!--break-->


