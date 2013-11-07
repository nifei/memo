# IPC

## Overview

主要使用命名通道来做进程间通信. 每个渲染进程和浏览器进程之间都有一个命名通道, 用来做异步通信. 

### 浏览器的IPC

浏览器进程的IPC在I/O线程里做. 来往消息都托管到`ChannelProxy`. 

### 渲染器IPC

渲染器的IPC在渲染器进程的主线程里, 其他事情放在其他线程里. 

## 消息

### 消息类型

按类型分, "routed" 和"control", 前者指定view, 后者是全局的. 请求view绘制一个区域是"routed", 剪切板是"control". 

按方向分, render->browser: ViewHost message; browser->view: View Message; 及PluginProcessHost & PluginProcess 两种message. 

### 消息声明

有参数的viewhost, routed消息
`IPC_MESSAGE_ROUTED2(ViewHostMsg_MyMessage, GURL, int)`

无参数的view, control消息
`IPC_MESSAGE_CONTROL0(ViewMsg_MyMessage)`

#### pickling values (参数序列化)

通过`ParamTraits`来做. 

### 发消息

用"channels"来发消息. Browser: `RenderProcessHost`使用通道, `RenderWidgetHost`调用`Send`. 传递消息指针, IPC层析构. Send的时候创建. 

### 处理消息

`IPC::Listener`会处理, 层层过滤, 有代码细节, 使用宏. 

### 安全问题

IPC注意问题. 

## 通道

`IPC::Channel`是通道主类. `IPC::SyncChannel`处理同步消息. 

通道非线程安全. 同步异步使用有区别. 浏览器主线程不应该被锁, 浏览器所有处理消息在I/O线程中. 

## 同步消息

带有返回值的消息需要同步. 

**注意:** ui线程不应该处理同步消息. 

### 声明同步消息

`IPC_SYNC_MESSAGE_CONTROL2_1`

### 发布同步消息

WebKit线程(在渲染器进程中, 非主线程)发出同步请求, 渲染器主线程会被分配到这个请求, WebKit线程会被block, 直到收到返回值. 

WebKit线程等待回复的时候, (渲染器)主线程继续接收浏览器进程的消息, 放在WebKit线程的消息列表队尾, 等到线程解锁再处理. 

### 同步消息处理细节

# Threading

## Overview

不鼓励线程安全对象和线程加锁. 尽量使用消息在线程间通信, 使用回调处理跨线程请求. 

## Existing Threads

BrowserProcess处理大多数线程. 
* ui_thread
* io_thread
* file_thread
* db_thread
* safe_browsing_thread

有的组件有自己的线程: 
* History
* Proxy service
* automation proxy

## Keeping the browser responsive

除了UI线程I/O线程也要少加锁, 因为会阻挠IPC. 

也尽量不要一个线程block另一个. 只有在数据立即可用时可以对线程加一下锁来做数据更新. [非要用的话](http://www.chromium.org/developers/lock-and-condition-variable). 

多数Chromium API是异步的, 使用回调或代理. 

## Getting stuff to other threads

### base::Callback<>, Async APIs, and Currying

这是线程间请求回调的细节. 

### PostTask

MessageLoop及细节

### base::Bind() and class methods

一个线程里的类调另一个线程里另一个类某个方法的方法. 

### How arguments are handled by base::Bind().

包装到InvokerStorage

## Callback cancellation

具体做法, 不要做什么. 

# Multi-process Resource Loading

## Background

Browser process 开启所有进程. 

## Overview

![alt text](http://www.chromium.org/_/rsrc/1220197833456/developers/design-documents/multi-process-resource-loading/Resource-loading.png "")

## WebKit

Chromium是怎么样把WebKit的`ResourceLoader`的`ResourceHandle`(实际执行请求的)的实现通过替换私有成员类`ResourceHandleInternal *d`的实现给改了的. 

`ResourceHandleInternal`实现一个回调接口, renderer会调用. 这个类和renderer通过`ResourceLoaderBridge`对话. 

## Renderer

renderer实现`ResourceLoaderBridge`使用单例(render process)`ResourceDispatcher`创建请求id, id传给`IPCResourceLoaderBridge`; 浏览器的回应包含同样的id, 再由Renderer转回WebKit用的. 

## Browser

`RenderProcessHost`会接收来自每个renderer的IPC请求, 请求传给`ResourceDispatcherHost`, 转成`URLRequest`对象. 这个对象产生的通知再一级一级传回给WebKit. 

## Cookies

`CookieMonster`处理. cookie是全局的. 

页面可以请求文档cookie. renderer会发同步请求给browser, webkit进程被挂起, 直到返回结果. 

[name](link "")

![alt text](link "title")
title