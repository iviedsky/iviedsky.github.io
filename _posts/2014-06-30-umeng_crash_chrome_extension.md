---
layout: post
title: 解析友盟iOS客户端崩溃日志 
category : 开发工具 
tagline: "Supporting tagline"
tags : [Chrome扩展应用, iOS]
markdown: rdiscount
---
{% include JB/setup %}
---

##解析友盟iOS客户端崩溃日志

###——Chrome扩展程序以及Python后台

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

![结构图](/image/umeng_extension_01.png =400x600)

1、当友盟统计错误详情页面打开以后，用户点击Chrome扩展应用时，该扩展应用会读取页面中Crash Log相关的信息，并通过HTTP POST请求发送至服务端。如下图所示，从页面中能够获取的主要信息包括：

a.标注1的部分，错误原因

b.标注2的部分，翻译前的日志

c.标注3的部分，包括dSYM的唯一标识、地址偏移量、基准地址等信息

![结构图](/image/umeng_extension_04.png =800x1200)


2、服务端根据从POST请求中接受到的dSYM唯一标识，找到存储在服务端的dSYM文件。并通过基准地址、地址偏移量等信息翻译日志内容，并将翻译后的内容返回。

3、Chrome扩展程序将服务端返回的内容写回到页面中。

------
###三、服务端：

####技术方案：

语言 - Python，HTTP服务器和后台框架 - Tornado

这一部分，博主参考了原作者的代码，并学习和进行了简单的优化。原代码Git链接如下：

此小节涉及的代码原址：https://github.com/Quotation/umeng-crash-symbol

####流程：

服务端根据从POST请求中接受到的dSYM唯一标识，找到存储在服务端的dSYM文的.并通过基准地址(Base Address)、地址偏移量(Slide Address)等信息翻译日志内容，并将翻译后的内容返回。

####主要技术点：

1、从dSYM文件中获取UDID

```python
def getDsymUDID(dsym):
    udid = subprocess.check_output("mdls -name com_apple_xcode_dsym_uuids -r    aw \"" + dsym + "\"", shell=True)
    return re.findall(r'".+?"', udid)[0][1:-1]
```

2、翻译-通过address，slide，baseAddr，arch从dSYM文件中获取symbol信息

```python
def getSymbol(dsymUDID, address, slide, baseAddr, arch):
    if dsymUDID not in symbolVersions:
        return None
    dir, app, _ = symbolVersions[dsymUDID]
    executable = getAppExecutable(app)
    realAddr = hex(int(address, 16) + int(slide, 16) - int(baseAddr, 16))
    os.chdir(dir)
    cmd = "xcrun atos -arch %s -o '%s'/'%s' %s" % (arch, app.decode("utf-8"), executable, realAddr)
    output = subprocess.check_output(cmd, shell=True)
    output = re.sub(r"\(in .+?\)\s", "", output).strip()
    return output
```

------
###四、Web客户端：

####技术方案：

语言 - JavaScript，平台 - Chrome Extension

当Chrome浏览器集成该扩展插件（Extension）后，若访问Umeng网站，则地址栏末尾会出现下图所示按钮。如当前页面为崩溃日志详情页面，则点击按钮后，触发翻译。
       
![翻译后](/image/umeng_extension_05.png =100x160)

####源代码：

1、manifest.json文件

```
{
  "manifest_version": 2,
  "name": "dYSMParser",
  "description": "dYSMParser",
  "version": "1.0.0",
  "background": {
    "scripts": ["background.js"],
    "persistent": false
  },
  "permissions": ["tabs", "http://*/*", "https://*/*"],
  "page_action": {
    "default_icon" : "icon.png",
    "default_title": "dYSMParser"
  },
  "content_scripts":[{
      "matches":["http://www.umeng.com/*"],
      "js":["jquery.min.js"]
  }]
}
```

2、background.js

```
function getDomainFromUrl(url){
     var host = "null";
     if(typeof url == "undefined" || null == url)
          url = window.location.href;
     var regex = /.*\:\/\/([^\/]*).*/;
     var match = url.match(regex);
     if(typeof match != "undefined" && null != match)
          host = match[1];
     return host;
}

function checkForValidUrl(tabId, changeInfo, tab) {
     if(getDomainFromUrl(tab.url).toLowerCase()=="www.umeng.com"){
          chrome.pageAction.show(tabId);
     }   
};

chrome.tabs.onUpdated.addListener(checkForValidUrl);

function handleClick(tab) {
                chrome.tabs.executeScript(tab.id, {file: "sendDSYM.js"})
}

chrome.pageAction.onClicked.addListener(handleClick);
```

3、sendDSYM.js

```
(function(){
                var report = $('#error-type-stack-trace-tab>pre').text();
                $.ajax({  
                                         url: "http://localhost:8000/convert",                                           type: "POST",
                                         cache: false,
                                         data: {crashreport: report},
                                         success: function(data){ 
                                                 $('#error-type-stack-trace-tab>pre').text(data)                                     
                                         },
                                         error: function(){
                                                 $('#error-type-stack-trace-tab>pre').text("Server error, Please connect to iviedsky@gmail.com to resolve this problem!")
                                         }
                });                          
})() 
```

------
###五、总结

这是一个简单的Client-Server工具类应用，能够方便iOS工作小组查看已发布版本的崩溃日志。

这个小工具涉及以下一些基础知识：

a.语言：python、javascript

b.框架以及开放平台：tornado、chrome extension

python是博主刚刚接触的一门语言，基础知识学习来源：

<a href="http://www.cnblogs.com/vamei/archive/2012/09/13/2682778.html" target="_blank" >python基础知识学习博客</a>

谷歌扩展，主要参考链接如下：

<a href="https://developer.chrome.com/extensions/getstarted.html" target="_blank">谷歌浏览器扩展插件官方文档</a>


