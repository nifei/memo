#Plugin Archicture

## Background

## Overview

插件是浏览器不稳定的主要因素. 因为插件是由第三方实现的我们无法控制其对系统的访问, 企图使用沙盒来隔离其所在进程是不现实的. 所以我们把插件都放在单独的进程里了. 

## Detailed design

### In-process plugins

Chromium可以在进程内和进程外运行插件. Chromium对插件的调用都始于WebKit层, WebKit层并不知道自己在单进程还是多进程中运行, WebKit层需要它的上层实现`WebKit::WebPlugin`接口, 用作和插件交互的接口. 这个接口被`WebPluginImpl`实现. `WebPluginImpl`继续和上层的`WebPluginDelegate`接口交流, 对进程内的插件, `WebPluginDelegateImpl`会实现`WebPluginDelegate`(译注: 对进程外的插件, `WebPluginDelegateProxy`会实现这个接口). `WebPluginDelegateImpl`会继续和NPAPI封装层交流. 

![alt text](http://www.chromium.org/_/rsrc/1270784736346/developers/design-documents/plugin-architecture/in_process_plugins.png "")

**历史遗留问题:**在WebKit层之前, `WebPluginImpl`是作为嵌入层的. `WebPluginImpl`通过`WebPluginDelegate`抽象接口和嵌入应用对话, `WebPluginDelegate`的实现根据进程内外而有所不同. 通过Chromium WebKit API的增补, 我们增加了`WebKit::WebPlugin`抽象接口, 它和`WebPluginDelegate`有相同的方法. **更好的设计**是把`WebPluginImpl`和`WebPluginDelegateImpl`合并在一起(在上图中有说, 这两个东西本来概念上做的就是同样的事情), 在实现`WebKit::WebPlugin`接口时区分进程内外. 在Chromium没有这样修改时因为代码太复杂了. 

## Out-of-process plugins

从上面那个图里虚线以下开始, Chromium对进程外的插件提供了不同的实现(译注:即对`WebPluginDelegate`的实现不同). 在虚线的地方分开, 在`WebPluginImpl`和`WebPluginDelegateImpl`中间插入了IPC层, 在两段的NPAPI代码可以复用. `WebPluginDelegateImpl`以上的代码, 和NPAPI层的代码现在在两个不同的进程中了. 

Render/Plugin交流通道的两端分别用`PluginChannel`和`PluginChannelHost`来表示. 有很多渲染进程, 每个插件有一个进程. 就是说渲染进程中对每一个用到的插件都有一个`PluginChannelHost`(AdobeFlash插件一个, Windows Media Player插件一个). 在每个插件进程中, 对每一个用到它的渲染进程都有一个`PluginChannel`. 

通道的每一端相应地映射一个插件的多个实例. 例如, 一个网页内嵌入两个Adobe Flash movies, 在渲染器一端有两个`WebPluginDelegateProxies`和别的, 在插件一端也有两个`WebPluginDelegateStub`和别的. 通道会负责这些对象在一个IPC连接内的多重通信. 

下面的图, 可以看见上面的进程内插件的图里出现过的部分灰掉, 在中间插入了新的进程外插件层. 

![alt text](http://www.chromium.org/_/rsrc/1270784866860/developers/design-documents/plugin-architecture/out_of_process_plugins.png "")

**历史问题:**以前我们使用stub/proxy模型来通信, 在IPC通道的两端各自有一个stub和一个proxy来收发一个插件的消息. 导致太多让人迷惑的类. 所以现在(上面的图), 在渲染进程一端, `WebPluginStub`被整合到`WebPluginDelegateProxy`类中, 这个类现在处理IPC通信渲染器一端所有的消息. 但是插件一端的stub和proxy没有结合, 还是两个类`WebPluginDelegateStub`和`WebPluginDelegateProxy`分别表示通信的两个方向, 虽然它们概念上是一个东西. 

## Windowless Plugins

无窗口插件是用来直接在渲染管道中运行的. WebKit想要通过插件在屏幕上绘制一个区域的时候会调用插件代码传入绘制上下文.  插件在页面上的绘制会有透明的时候常使用无窗口插件, 插件的绘制和页面本来的像素会融合在一起. 

在进程之外使用无窗口插件的时候, 你仍然需要把插件的渲染(同步地)整合到WebKit的渲染通道中. 一个本来很慢的方法是, 把插件要绘制的区域剪裁下来, 同步地传递给插件进程取绘制. 通过共享内存可以加速这个方法. 

但是渲染速度就此受到插件进程的限制(想象下有30个透明插件的话, 我们就得往返插件进程30次). 所以我们不这么干, 我们让插件异步绘制, 就像已经存在的页面异步渲染到屏幕上一样. 绘制时, 渲染器有一个插件已经渲染好的后备存储, 渲染器先显示这个, 等插件发来异步的更新时再更新这块区域. 

这些都被"透明"插件给弄复杂了. 插件进程需要知道哪些像素是它要覆盖掉的. 所以插件会缓存一份渲染器最后一次发来的内容作为背景, 然后在这个背景上反复绘制. 

总结下, 下面是和无窗口插件绘制的那块插件相关的缓存: 

渲染器进程
 - 插件上一次渲染的后备存储
 - 和插件共享的, 用来接收更新的共享内存("运输DIB")
 - 插件背后的页面背景的备份(下面讲)
 
插件进程
 - 插件后面的背景的备份, 用做绘制时的源
 - 和渲染器共享的共享内存, 用来发送更新("运输DIB")

为什么渲染器也要保留一份页面背景的备份呢? 如果页面背景变化了, 我们需要同步地让插件重新在新的背景上绘制(本来是异步绘制的, 就会滞后了). 我们可以通过比较新的背景和备份得到更新的内容. 因为插件和渲染进程是异步的, 它们需要各自的备份. 

## Overall system

下面的概括图包含两个渲染进程, 和同一个共享的, 进程之外的插件进程通信. 一共有三个插件实例. 注意图里的WebPluginStub和WebPluginDelegateProxy已经合并到一起了, 图过期了. 

![alt text](http://www.chromium.org/_/rsrc/1220197834023/developers/design-documents/plugin-architecture/pluginsoutofprocess.png "")

译注: 官方文档其实写的很简单.. 我觉得这个讲的更清楚: http://blog.csdn.net/milado_nju/article/details/7216136

NPAPI一方的实现也有讲. 
