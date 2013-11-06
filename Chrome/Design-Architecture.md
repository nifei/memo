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

通常每个新的窗口或tab会开启一个新进程. 浏览器会创建新进程并指导它创建单独的`RenderView`. 

有时候会有必要在tab和窗口之间共享渲染进程. web应用会打开一个新窗口并且和它同步通信, 例如在javascript中使用`window.open`. 这种情况下打开一个新窗口或tab需要重用已经打开的窗口的进程. 进程数目太多的时候, 或者用户已经有一个进程打开那个域名的时候, 我们也会使用策略把新tab分配给已经存在的进程. 这些策略在这里Process Models<http://www.chromium.org/developers/design-documents/process-models>. 

## 检测崩溃或者出错的渲染器

每个浏览器进程的IPC连接都会监视进程句柄. 如果这些句柄是signaled, 渲染器进程崩溃了, 那么tabs就会知道. 现在我们显示"sad tab"来告诉用户渲染器崩溃了. 页面可以通过Reload键或者再跳转一次来重新加载. 这时会发现没有进程, 然后创建一个. 

## 渲染器的沙盒

(Sandboxing<http://baike.baidu.com/view/514384.htm>)

既然Webkit是在独立进程中运行的我们就有机会限制它对系统资源的访问. 例如我们可以保证渲染器只通过它的父浏览器访问网络. 雷斯蒂, 我们可以使用host操作系统的内置权限限制它对文件系统的访问. 

除了限制渲染器对文件系统和网络的访问, 我们还可以限制它对用户输出和相关对象的访问. 每个渲染进程运行在独立的, 用户看不见的Windows "Desktop"中. 这就避免了渲染器对打开新窗口或者捕捉键盘消息的妥协(不懂). 

## Giving back memory (恢复记忆?)

因为渲染器在单独进程中运行, 直接结果就是降低隐藏的tab的优先级. 通常Windows上最小化的进程会自动把内存放在"available memory"的内存池中. 内存不足的情况下, Windows会先把这块内存swap到硬盘上来保证用户可见进程的响应. 我们对隐藏tab采用相同的原则. 渲染进程没有top-level tab的时候, 我们可以提示系统, 如有必要这个进程可以先释放"working set"那么多的内存swap到硬盘上. (we can release that process's "working set" size as a hint to the system to swap that memory out to disk first if necessary.) 因为发现减少working set size会影响用户跳转tab的性能, 我们要一点点释放这块内存. 这意味着如果用户跳转回一个最近使用的Tab, 这个tab的内存会比更早使用的tab的内存更有可能在内存中(being paged, not on disk). 内存足够运行所有程序的用户不会注意到这个进程: Windows只会在需要的时候才真正取回数据, 所以有足够内存的话不会有性能问题. (Windows will only actually reclaim such data if it needs it, so there is no performance hit when there is ample memory.)

在低内存的情况下这帮助我们更好地优化内存足迹. 较少使用的后台页面的内存全都被swap(到硬盘上)了, 前台页面的数据全在内存中. 相比之下, 单进程的浏览器上所有页面的数据随机分布在内存中, 无法清除第分离使用的不使用的数据, 既浪费内存性能也不好. 

## plug-ins

Firefox风格的NPAPI plug-in在各自的进程中运行, 和渲染器分开. 详见Plugin-Architecute<http://www.chromium.org/developers/design-documents/plugin-architecture>

![alt text](link "title")
title