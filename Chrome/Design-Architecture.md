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

http://www.chromium.org/_/rsrc/1220197832277/developers/design-documents/multi-process-architecture/arch.png