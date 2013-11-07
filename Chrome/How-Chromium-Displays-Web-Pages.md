# Chromium如何显示页面

http://www.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome

文档从下向上讲述了Chromium怎样显示网页. 确保你看过[multi-process architecture](http://www.chromium.org/developers/design-documents/multi-process-architecture)设计文档了. 理解主要组件的阻塞流程很重要. [multi-process resource loading](http://www.chromium.org/developers/design-documents/multi-process-resource-loading)这一篇讲怎样从互联网上获取页面可能也有帮助. 

## 概念上的应用层

![alt text](https://docs.google.com/drawings/pub?id=1gdSTfvLxbJDbX8oiWo5LTwAmXmdMQvjoUhYEhfhj0-k&w=473&h=211 "(The original Google Doc for this illustration is http://goo.gl/MsEJX which is open for editing by any @chromium.org)")
(The original Google Doc for this illustration is http://goo.gl/MsEJX which is open for editing by any @chromium.org)

每个方块代表一个应用层. 应用层不应该知道或者依赖更高层的应用. 

* **Webkit:** 在Safari, Chromium和其他基于Webkit的浏览器中使用的渲染引擎. **Port**是Webkit的一部分, 它整合平台相关的系统服务, 像资源加载或者图片什么的. 

* **Glue:** Webkit类型到Chromium类型的转换. 这是"Webkit嵌套层". 这是Chromium和test_shell(测试Webkit用的)这两个浏览器的基础. 

* **Renderer/Render Host:** Chromium的"多进程嵌入层". 它在进程边检之间代理通知和指令. 

* **WebContents:** 内容模块的可重用组件, 主类. 可嵌套, 允许Html到view的多进程渲染. 详见[Content module pages](http://www.chromium.org/developers/content-module). 

* **Brwoser:** 浏览器窗口, 包括多个WebContentses. 

* **Tab Helpers:** 可以附加到WebContents上的独立对象(通过WebContentsUserData mixin). 浏览器往它持有的WebContentses(一个网站图标, 一个信息栏, 等)上附加一个这些玩意的混搭. 

## Webkit

我们用[Webkit](http://webkit.org/)开源工程来排版网页. 代码从Apple拉过来放在路径`/third_party/WebKit`下. Webkit主要是由代表核心排版功能的"WebCore"和运行JavaScript的"JavaScriptCore"组成. 我们只用JavaScriptCore来跑测试, 通常使用性能更好的V8 JavaScrip引擎来代替它. 我们实际上并不使用苹果称为"WebKit"的那一层, 那个是WebCore和诸如Safari的OS X应用的嵌入式API. 为了方便我们一般把苹果的代码称作"WebKit". 

### The WebKit port

在最底层我们有我们的WebKit "port". 这是需要的特定平台和平台独立的WebCore代码交互的功能的实现. 文件在WebKit tree中, 通常在`chromium`路径或者Chromium后缀的文件. 端口(port)的大部分不限定OS: 可以把它想成WebCore的"Chromium端口". 有些部分例如字体渲染必需根据平台分别处理. 

* 网络通信是由[multi-process resource loading](http://www.chromium.org/developers/design-documents/multi-process-resource-loading)系统处理的, 而不是从渲染进程直接移交操作系统. 

* 图片使用Android开发的Skia图片库. 这个跨平台的图片库处理所有图片和图片权限, 除了文字. Skia放在`/third_party/skia`下. 图片操作的切入点是`/webkit/port/platform/graphics/GraphicsContextSkia.cpp`. 它会用到同路径下和`/base/gfx`下的其他文件.

## The WebKit glue

Chromium应用使用和第三方WebKit代码不同的类型, 编码风格和代码排版. WebKit "glue" 使用Google代码风格和类型提供了更方便的嵌入API(例子, 用`std::string`代替`WebCore::string`, `GURL`代替`KURL`). 黏合代码(glue code)置于`/webkit/glue`. 黏合对象和WebKit对象命名通常类似, 以"Web"打头. 例如`WebCore::Frame`变成`WebFrame`. 

WebKit"黏合"层把Chromium代码基础的其他部分和WebCore的数据类型隔离开来, 将WebCore的变化对Chromium代码基础(基线?)的影响降到最小. Chromium从不直接使用WebCore的数据类型. 加到WebKit的API在需要调用某个WebCore对象的时候为了Chromium黏合. 

"test shell"应用是个用于测试WebKit端口和黏合代码的极简的浏览器. 它像chromium一样使用相同的glue接口和WebKit通信. 开发者不必使用复杂的浏览器特性, 线程和进程就可以测试新代码. 这个应用也用来跑WebKit的自动化测试. 但是"test shell"不像Chromium一样以多进程的方式使用WebKit. 内容模块被放置在一个叫做"content shell"的应用中立即跑测试. 

## 渲染进程

![alt text](http://www.chromium.org/_/rsrc/1342584100941/developers/design-documents/displaying-a-web-page-in-chrome/Renderingintherenderer-v2.png "")

Chromium的渲染进程使用黏合接口嵌入在WebKit端口中. 代码不多: 主要任务是作为连接浏览器的IPC通道的渲染器端. 

渲染器中最重要的类是`RenderView`, 在`/content/renderer/render_view_impl.cc`. 这个对象代表网页. 它处理所有来往浏览器进程的跳转相关的指令. 它继承自`RenderWidget`, `RenderWidget`提供绘制和输入事件处理. `RenderView`通过全局的`RenderProcess`对象(每个渲染进程)和浏览器进程通信. 

**FAQ:RenderWidget和RenderView的区别?** `RenderWidget`通过实现黏合层叫做`WebWidgetDelegate`的抽象接口映射到`WebCore::Widget`对象. 基本上就是一个接收输入事件,我们要往上绘制的窗口. `RenderView`继承自`RenderWidget`, 是窗口或页面的内容. `RenderView`除了绘制和接收输入事件之外还要处理跳转指令. 只有一种情况下一个`RenderWidget`不是一个`RenderView`: 网页上的选择框. 那种有一个向下的箭头, 会弹出来一个选择列表的框框(combo box). 选择框需要使用原生窗口渲染以保证在最顶层, 还可能超过窗口边界. 这样的窗口也需要绘制和接收事件但是并不是一个"网页"(`RenderView`). 

### 渲染器的线程

一个渲染器有两个线程(见[multi-process architecture](http://www.chromium.org/developers/design-documents/multi-process-architecture)的流程图, 或者见[threading in Chromium](http://www.chromium.org/developers/design-documents/threading)怎样编程). 主要对象像`RenderView`和所有的WebKit 代码会跑在渲染线程中. 在所有其他事情之中这允许我们同步地从渲染器向浏览器发送消息. 少数几个操作要求来自浏览器的结果要继续进行. 一个例子是JavaScript要求的时候获取页面cookie. 渲染器线程会阻塞, 主线程排队处理所有的消息直到找到正确的回应. 这期间受到的所有消息会按顺序放在渲染器线程的后面. 

## 浏览器进程

![alt text](http://www.chromium.org/_/rsrc/1220197831473/developers/design-documents/displaying-a-web-page-in-chrome/rendering%20browser.png "")

### 底层浏览器进程对象

所有和渲染器进程之间的[IPC](http://www.chromium.org/developers/design-documents/inter-process-communication "")通信都是在浏览器的I/O线程中完成的. 这个线程还处理避免介入UI的[network communication](http://www.chromium.org/developers/design-documents/multi-process-resource-loading ""). 

`RenderProcessHost`在主线程(UI线程)中初始化的时候会创建新的渲染器进程和带有通到渲染器的命名的管道`ChannelProxy`IPC对象. 这个对象(ChannelProxy)在浏览器的I/O线程中运行, 监听渲染器的通道, 自动把所有消息发送回UI线程的`RenderprocessHost`. 这个通道(ChannelProxy)上会安装一个`ResourceMessageFilter`, 滤掉能在I/O线程中直接处理的常规信息像网络请求什么的. 过滤的实现在`ResourceMessageFilter::OnMessageReceived`. 

UI线程的`RenderProcessHost`负责分配每个view的信息到合适的`RenderViewHost`(介个会自己处理一些和view没有关系的信息)去. 消息分配实现在`RenderProcessHost::OnMessageReceived`. 

### 上层浏览器进程对象

具体的view的消息会被传到`RenderViewHost::OnMessageReceived`. 多数消息在此处理, 其他的在基类`RenderWidgetHOST`中处理. 这两个对应渲染器中的`RenderView`和`RenderWidget`(见上面的"The Render Process"). 每个平台有一个view的类(`RenderWidgetHostView[Aura|Gtk|Mac|Win]`)来实现原生的view系统的整合. 

在`RenderView/Widget`上层是`WebContents`对象, 多数消息最终由这个对象的一个函数调用来处理. `WebContents`表现网页内容. 是内容模块的顶层对象, 负责在一个矩形视图中显示网页. 详见[content module](http://www.chromium.org/developers/content-module ""). 

`WebContents`对象包含在`TabContentsWrapper`中. 在`chrome/`下面, 负责一个tab. 

## 说明用例子

其他的跳转和启动的例子在[Getting Around the Chromium Source Code](http://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code)中. 

### "set cursor"消息的生命周期

设置鼠标是从渲染器发送到浏览器的典型消息的一个例子. 在渲染器中, 发生了这些事情. 

* Set cursor 消息由WebKit内部生成, 通常是用来相应输入事件. Set cursor消息始于`RenderWidget::SetCursor in content/renderer/render_widget.cc`. 

* (Set Cursor消息)会调用`RenderWidget::Send`来分配消息. `RenderView`也用这个方法来向浏览器发送消息, 它会调用`RenderThread::Send`. 

* 会调到`IPC::SyncChannel`, 函数内部会代理发送这个消息到渲染器的主线程, 放在浏览器命名通道的后面. 

然后浏览器会接管过来: 

* `RenderProcessHost`中的`IPC::ChannelProxy`会接收到所有浏览器I/O线程的消息. 它会先把消息发送到`ResourceMessageFilter`分派网络请求和I/O线程的直接相关消息. 因为Set Cursor这个消息不会被过滤掉, 它会继续发送到浏览器的UI线程(`IPC;ChannelProxy`会在内部处理). 

*  在`content/browser/renderer_host/render_process_host_impl.cc`中`RenderProcessHost::OnMessageReceived`拿到相应渲染进程的所有view的消息. 它会直接处理几种类型的消息, 剩下的分发到和发送消息的`RenderView`相应的`RenderViewHost`上. 

* `RenderViewHost`没有处理的消息都会发送到`RenderWidgetHost`中处理, 包括Set cursor消息. 

* `content/browser/renderer_host/render_view_host_impl.cc`中的消息表最终在`RenderWidgetHost::OnMsgSetCursor`中收到消息, 调用相应的UI函数来设置鼠标. 

### "mouse click"消息的周期

发送鼠标点击是从浏览器发送到渲染器的典型消息. 

* 浏览器的UI线程通过`RenderWidgetHostViewWin::OnMouseEvent`收到Windows消息, 然后调用同一个类里的`ForwardMouseEventToRenderer `. 

* 发送函数将输入事件打包成一个跨平台的`WebMouseEvent`最后发送到相关的`RenderWidgetHost`. 

* `RenderWidgetHost::ForwardInputEvent`创建一个IPC消息`ViewMsg_HandleInputEvent`, 转播`WebInputEvent`给它, 然后调用`RenderWidgetHost::Send`. 

* 这样做只是转播`RenderProcessHost::Send`函数, `RenderProcessHost::Send`会按顺序发送消息到`IPC::ChannelProxy`. 

* 在内部`IPC::ChannelProxy`把消息托管发送到浏览器的I/O线程, 并且写到对应渲染器的命名管道中. 

注意`WebContents`会创建许多其他类型的消息, 特别是跳转一类的. 这些消息通过类似的途径从`WebContents`传递到`RenderViewHost`. 

渲染器接管: 

* 渲染器主线程中的`IPC::Chanel`读取浏览器发送的消息, `IPC::ChannelProxy`把消息委托到渲染器线程. 

* `RenderView::OnMessageReceived`拿到消息. 很多种消息在此直接处理. click消息不是, 它会(和其他未被处理的消息一起)发送到`RenderWidget::OnMessageReceived`, 后者按顺序发送到`RenderWidget::OnHandelInputEvent`. 

* 输入事件被发送到`WebWidgetImpl::HandleInputEvent`, `WebWidgetImpl::HandleInputEvent`把它转换成一个WebKit`PlatformMouseEvent`类型然后发送给WebKit离得`WebCore::Widget`类. 
