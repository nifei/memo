# Chromium如何显示页面

http://www.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome

文档从下向上讲述了Chromium怎样显示网页. 确保你看过multi-process architecture<http://www.chromium.org/developers/design-documents/multi-process-architecture>设计文档了. 理解主要组件的阻塞流程很重要. multi-process resource loading<http://www.chromium.org/developers/design-documents/multi-process-resource-loading> 这一篇讲怎样从互联网上获取页面可能也有帮助. 

## 概念上的应用层

![alt text](https://docs.google.com/drawings/pub?id=1gdSTfvLxbJDbX8oiWo5LTwAmXmdMQvjoUhYEhfhj0-k&w=473&h=211 "(The original Google Doc for this illustration is http://goo.gl/MsEJX which is open for editing by any @chromium.org)")
(The original Google Doc for this illustration is http://goo.gl/MsEJX which is open for editing by any @chromium.org)

每个方块代表一个应用层. 应用层不应该知道或者依赖更高层的应用. 

* **Webkit:** 在Safari, Chromium和其他基于Webkit的浏览器中使用的渲染引擎. **Port**是Webkit的一部分, 它整合平台相关的系统服务, 像资源加载或者图片什么的. 

* **Glue:** Webkit类型到Chromium类型的转换. 这是"Webkit嵌入层". 这是Chromium和test_shell(测试Webkit用的)这两个浏览器的基础. 

* **Renderer/Render Host:** Chromium的"多进程嵌入层". 它在进程边检之间代理通知和指令. 

* **WebContents:** 内容模块的可重用组件, 主类. 可嵌入, 允许Html到view的多进程渲染. 详见Content module pages<http://www.chromium.org/developers/content-module>. 

* **Brwoser:** 浏览器窗口, 包括多个WebContentses. 

* **Tab Helpers:** 可以附加到WebContents上的独立对象(通过WebContentsUserData mixin). 浏览器往它持有的WebContentses(一个网站图标, 一个信息栏, 等)上附加一个这些玩意的混搭. 

## Webkit

我们用Webkit<http://webkit.org/>开源工程来排版网页. 代码从Apple拉过来放在路径`/third_party/WebKit`下. Webkit主要是由代表核心排版功能的"WebCore"和运行JavaScript的"JavaScriptCore"组成. 我们只用JavaScriptCore来跑测试, 通常使用性能更好的V8 JavaScrip引擎来代替它. 我们实际上并不使用苹果成为"WebKit"的那一层, 那个是WebCore和诸如Safari的OS X应用的嵌入式API. 为了方便我们一般把苹果的代码称作"WebKit". 

### The WebKit port

在最底层我们有我们的WebKit "port". 这是需要的特定平台和平台独立的WebCore代码交互的功能的实现. 文件在WebKit tree中, 通常在`chromium`路径或者Chromium后缀的文件. 端口(port)的大部分不限定OS: 可以把它想成WebCore的"Chromium端口". 有些部分例如字体渲染必需根据平台分别处理. 

* 网络通信是由multi-process resource loading<http://www.chromium.org/developers/design-documents/multi-process-resource-loading>系统处理的, 而不是从渲染进程直接移交操作系统. 

* 图片使用Android开发的Skia图片库. 这个跨平台的图片库处理所有图片和图片权限, 除了文字. Skia放在`/third_party/skia`下. 图片操作的切入点是`/webkit/port/platform/graphics/GraphicsContextSkia.cpp`. 它会用到同路径下和`/base/gfx`下的其他文件.

## The WebKit glue

Chromium应用使用和第三方WebKit代码不同的类型, 编码风格和代码排版. WebKit "glue" 使用Google代码风格和类型提供了更方便的嵌入API(例子, 用`std::string`代替`WebCore::string`, `GURL`代替`KURL`). 黏合代码(glue code)置于`/webkit/glue`. 黏合对象和WebKit对象命名通常类似, 以"Web"打头. 例如`WebCore::Frame`变成`WebFrame`. 

WebKit"黏合"层把Chromium代码基础的其他部分和WebCore的数据类型隔离开来, 将WebCore变化对Chromium代码基础(基线?)的影响降到最小. Chromium从不直接使用WebCore的数据类型. 加到WebKit的API在需要调用某个WebCore对象的时候为了Chromium黏合. 

"test shell"应用是个用于测试WebKit端口和黏合代码的极简的浏览器. 它像chromium一样使用相同的glue接口和WebKit通信. 开发者不必使用复杂的浏览器特性, 线程和进程就可以测试新代码. 这个应用也用来跑WebKit的自动化测试. 但是"test shell"不像Chromium一样以多进程的方式使用WebKit. 内容模块被放置在一个叫做"content shell"的应用中立即跑测试. 

### 渲染程序

![alt text](http://www.chromium.org/_/rsrc/1342584100941/developers/design-documents/displaying-a-web-page-in-chrome/Renderingintherenderer-v2.png "")

Chromium的渲染进程使用黏合接口嵌入在WebKit端口中. 代码不多: 主要任务是作为连接浏览器的IPC通道的渲染器端. 

渲染器中最重要的类是`RenderView`, 在`/content/renderer/render_view_impl.cc`. 这个对象代表网页. 它处理所有来往浏览器进程的跳转相关的指令. 它继承自`RenderWidget`, `RenderWidget`提供绘制和输入事件处理. `RenderView`通过全局的`RenderProcess`对象(每个渲染进程)和浏览器进程通信. 

**FAQ:RenderWidget和RenderView的区别?** `RenderWidget`通过实现黏合层叫做`WebWidgetDelegate`的抽象接口映射到`WebCore::Widget`对象. 基本上就是一个接收输入事件,我们要往上绘制的窗口. `RenderView`继承自`RenderWidget`, 是窗口或页面的内容. `RenderView`除了绘制和接收输入事件之外还要处理跳转指令. 只有一种情况下一个`RenderWidget`不是一个`RenderView`: 网页上的选择框. 那种有一个向下的箭头, 会弹出来一个选择列表的框框(combo box). 选择框需要使用原生窗口渲染以保证在最顶层, 还可能超过窗口边界. 这样的窗口也需要绘制和接收事件但是并不是一个"网页"(`RenderView`). 

### 渲染器的线程

一个渲染器有两个线程(见multi-process architecture<http://www.chromium.org/developers/design-documents/multi-process-architecture>的流程图, 或者见threading in Chromium<http://www.chromium.org/developers/design-documents/threading>怎样编程). 主要对象像`RenderView`和所有的WebKit 代码会跑在渲染线程中. 在所有其他事情之中这允许我们同步地从渲染器向浏览器发送消息. 少数几个操作要求来自浏览器的结果要继续进行. 一个例子是JavaScript要求的时候获取页面cookie. 渲染器线程会阻塞, 主线程排队处理所有的消息直到找到正确的回应. 这期间受到的所有消息会按顺序放在渲染器线程的后面. 

## 浏览器进程

![alt text](http://www.chromium.org/_/rsrc/1220197831473/developers/design-documents/displaying-a-web-page-in-chrome/rendering%20browser.png "")

### 底层浏览器进程对象

所有和渲染器进程之间的[IPC](http://www.chromium.org/developers/design-documents/inter-process-communication "")通信都是在浏览器的I/O线程中完成的. 这个线程还处理避免介入UI的[network communication](http://www.chromium.org/developers/design-documents/multi-process-resource-loading ""). 

`RenderProcessHost`在主线程(UI线程)中初始化的时候会新的渲染器进程和带有通到渲染器的命名的管道`ChannelProxy`IPC对象. 这个对象(ChannelProxy)在浏览器的I/O线程中裕兴, 监听渲染器的通道, 自动把所有消息发送回UI线程的`RenderprocessHost`. 这个通道(ChannelProxy)上会安装一个`ResourceMessageFilter`, 滤掉能在I/O线程中直接处理的常规信息像网络请求什么的. 过滤的实现在`ResourceMessageFilter::OnMessageReceived`. 

UI线程的`RenderProcessHost`负责分配每个view的信息到合适的`RenderViewHost`(介个会自己处理一些和view没有关系的信息)去. 消息分配实现在`RenderProcessHost::OnMessageReceived`. 

### 上层浏览器进程对象

具体的view的消息会被传到`RenderViewHost::OnMessageReceived`. 多数消息在此处理, 其他的在基类`RenderWidgetHOST`中处理. 这两个对应渲染器中的`RenderView`和`RenderWidget`(见上面的"The Render Process"). 每个平台有一个view的类(`RenderWidgetHostView[Aura|Gtk|Mac|Win]`)来实现原生的view系统的整合. 

在`RenderView/Widget`上层是`WebContents`对象, 多数消息最终由整个对象的一个函数调用来处理. `WebContents`表现网页内容. 是内容模块的顶层对象, 负责在一个矩形视图中显示网页. 详见[content module](http://www.chromium.org/developers/content-module ""). 

`WebContents`对象包含在`TabContentsWrapper`中. 在`chrome/`下面, 负责一个tab. 

## 说明用例子

其他的跳转和启动的例子在[Getting Around the Chromium Source Code](http://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code)中. 

[name](link "")

![alt text](link "title")
title