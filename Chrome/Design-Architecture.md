# 多进程架构
http://www.chromium.org/developers/design-documents/multi-process-architecture

这个文档讲了Chrominum的高级架构.

## 问题(这段不用看)

完美的渲染引擎很难做, 崩溃, 冻结, 不安全. 

浏览器现状: 有点像过去单用户, 多任务协作的操作系统. 在过去一个应用可以拖垮整个操作系统, 现在一个页面也能拖垮整个浏览器(这有点像Bolt引擎设计的理念, 界面出错了不会拖垮整个应用). 只要浏览器或者plug-in出了一点错误, 整个浏览器所有的页面都完蛋了. 

现在的操作系统更健壮, 每个应用程序都和其他的应用程序隔离开来在单独的进程中. 一个应用崩溃了不会影响其他的应用乃至操作系统, 用户之间的数据访问也被限制了. 

## 架构总览

我们在渲染引擎中对每个浏览器页面使用独立的进程来避免应用被小错误破坏. 渲染引擎进程之间不允许互相访问, 渲染引擎进程也不能访问操作系统. 某种程度上这个给浏览器带来了操作系统级别的内存保护和访问控制. 

运行UI和惯例页面和plugin的主进程叫做"浏览器进程"或者"浏览器"(browser process & browser). 页面的进程叫做"渲染进程"或者"渲染器"(render process & rnderer). 渲染器使用Webkit<http://webkit.org/>开源布局引擎阿狸渲染和布局HTML. 

![alt text](http://www.chromium.org/_/rsrc/1220197832277/developers/design-documents/multi-process-architecture/arch.png "到底是进程啊还是线程啊还有为什么有的Render有一个view有的有两个啊")
架构图

## 管理渲染进程

每个渲染进程有个全局的`RenderProcess`对象来管理和父浏览器进程之间的通信并持有全局状态. 浏览器进程对应每个view和render持有一个`RenderProcessHost`. 每个view有一个view Id用来区别于其他使用同一个渲染器的view. 在渲染器内部view id是唯一的但是浏览器内不是, 所以确定一个view需要一个`RenderProcessHost`和view id. 浏览器和每个具体的tab上的内容之间的通信是通过这些`RenderViewHost`对象来完成的, `RenderViewHost`对象知道如何通过它们的`RenderProcessHost`来往`RenderProcess`和`RenderView`发送消息. 

## 组件和接口

在渲染进程中:

* `RenderProcess`处理和浏览器中的`RenderProcessHost`之间的IPC. 每个渲染进程有一个`RenderProcess`. 所有浏览器<->渲染器通信都是这样做的. 

* `RenderView`对象和它对应的`RenderViewHost`在浏览器进程中通信(通过`RenderProcess`), 在WebKit嵌入层中. 这个对象(RenderView)代表一个tab或窗口上的web page. 

在浏览器进程中: 

* `Browser`对象代表一个top-level的浏览器窗口
* `RenderProcessHost`对象代表单个浏览器<->渲染器IPC通信之中浏览器一方. 在浏览器进程中每个渲染进程对应一个`RenderProcessHost`. 
* `RenderViewHost`对象封装和远端`RenderView`的通信, `RenderWidgetHost`处理浏览器中`RenderWidget`的输入和绘制. 

更多细节见How Chromium displays web pages <http://www.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome>. 

## 共享渲染进程




![alt text](link "title")
title