原文: http://aosabook.org/en/posa/high-performance-networking-in-chrome.html

## 网络资源请求的生命周期

W3C [Navigation Timing specification](http://www.w3.org/TR/navigation-timing/)提供了一个浏览器API观察每个请求生命周期的性能数据. 我们来观察一下这些模块, 因为每一个都在提供良好的用户体验上起着至关重要的作用. 

![alt text](http://www.aosabook.org/en/posa/chrome-images/navtiming.png "图1.1 - 跳转时间")

给定一个网络资源的URL, 浏览器首先会检查它本地和应用的缓存. 如果你之前获取过这个资源并提供了合适的缓存头([appropriate cache headers](http://www.aosabook.org/https://developers.google.com/speed/docs/best-practices/caching), `Expires`, `Cache-Control`, etc.), 就可能可以使用本地缓存来填充请求 ---- 最快的请求就是不做请求. 不然的话, 要是我们必须重新验证资源, 或者它过期了, 或者没有见过它, 那就必须来一次昂贵的网络请求了. 

有了hostname和资源路径, Chrome首先会在它可以重用的连接中查找 -- 以`{scheme, host,port}`存储的套接字们. 也可以这样, 如果你配置了代理, 或者指定了[proxy auto-config](http://www.aosabook.org/http://en.wikipedia.org/wiki/Proxy_auto-config)(PAG)脚本, chrome就会通过适当的代理查找连接. PAC脚本允许根据URL或其他指定的规则使用不同的代理, 每种都可以它们自己的套接字池. 最后要是上面的条件都不符合, 就只能以解析主机名到IP地址来开始这个请求了 -- 也就是要先做DNS查找. 

运气好的话, 主机名已经在缓存中, 也就是说系统查找一下就可以返回. 否则的话, 在发出DNS查询之前什么都做不了. DNS查询的时间和你的网络供应商, 网站知名度, 域名和中间节点相似度, 以及被这个域名授权的服务器的反应速度都有关系. 也就是说变数很大, 花上几百毫秒也不奇怪. 

![alt text](http://www.aosabook.org/en/posa/chrome-images/three-way.png "图1.2 三次握手")

有了IP地址, Chrome就可以打开一个新的TCP连接, 意味着我们得执行"三次握手": `SYN > SYN-ACK > ACK`. 每个新的TCP连接都会增加这么一个来回, 两边都有潜在的延时 -- 不能绕过去. 根据客户端和服务器的距离, 以及选择的路由路径, 这可能产生从几十到几千毫秒不等的延迟. 而这时候, 还没有一个有用的字节在传输呢. 

TCP握手结束后如果我们要连接到一个安全目标(HTTPS)上, 需要执行SSL握手. 这可能在客户端和服务器之间多跑两个来回. 如果SSL session在缓存中, 我们只能少跑一个来回. 

终于Chrome能发送HTTP请求了(图1.1 `requestStart`). 服务器一收到请求就可以执行它然后以流的形式向客户端发送返回数据. 这会用掉最少一个来回的时间加上服务器处理请求的时间. 到这里我们的事情做完了 -- 除非实际的回复是一个HTTP重定向, 那样的话我们就得把这个流程再跑一遍. 你的页面上有几个免费的重定向? 好好考虑一下吧. 

把所有的延时都算进去了吗? 为了看清楚现在的问题, 假设一下常见带宽连接下最坏的情况: 本地未缓存, 相对快的DNS查找(50ms), TCP握手, SSL协商, 相对快的(100ms)服务器响应, 80ms一个来回(RTT)(USA平均).

 * DNS 50ms
 * TCP 握手 80ms (one RTT)
 * SSL 握手 160ms (two RTT)
 * 请求服务器 40ms
 * 服务器处理 100ms
 * 服务器响应 40ms

单次请求用掉470毫秒, 比实际上服务器的响应时间(250ms)多了80% -- 我们在这里做了些事情. 实际上470毫秒已经是乐观估计了: 

 * 如果服务器的返回数据比TCP congestion window大(4-15kb), 就会有更多的来回传输. 
 * 如果我们需要获取丢失的验证或者执行在线验证状态检查(OCSP)的话, SSL延迟会更严重, 这两种情况都需要一次新的TCP连接, TCP连接会用掉几百甚至上千毫秒. 
 
## 怎么样才叫"快"?
 
 在上面的例子里, DNS的网络开销, 握手和来回次数都增加了总时间 -- 服务器响应时间只在总延时里占20%. 但是在庞大的体系下, 这点延时要紧吗? 如果你读到了这里我相信你已经知道答案了: 非常重要. 
 
 过去的用户体验调研给出了我们作为用户, 对线上和线下应用的响应时间, 有一致的感受: 
 
表1.1 - 用户对延时的感受
|Delay|User Reaction|
|:---------:|:---------:|
|0 - 100 ms	|Instant
|100 - 300 ms	|Small perceptible delay
|300 - 1000 ms	|Machine is working
|1 s+	|Mental context switch
|10 s+	|I’ll come back later…

表1.1也解释了对网站性能评价的潜规则(O.O?): 渲染页面, 至少在250ms内给点可见的反馈吸引住用户. 这并非为了单纯滴为了速度而追求速度. 对Google, Amazon, Microsoft和其他上千个网站的调研表明额外的延时对网站底线有直接的影响: 更快的网站可以有更多的页面被看见, 和用户的契约程度更高, 带来更高的转化率($_$)

所以我们最大的延时需要控制在250ms以内, 在上面的例子我们看到DNS查询, TCP和SSL握手以及请求的传输加起来多达370ms. 我们已经超过预算50%了, 这还没把服务器处理时间算在内呢. 

对多数用户和web开发者来说, DNS, TCP和SSL延时是完全透明的在网络层处理的, 很少去想到. 但是这些步骤对用户体验来说都是关键的, 因为每一个额外的网络请求都会增加几十到几百毫秒的延迟. 这也是为什么Chrome的network stack不仅仅是一个简单的socket handler. 

现在我们定位到问题了, 来看具体实现吧. 

## Chrome’s Network Stack from 10,000 Feet

### 多进程架构

略

![alt text](http://www.aosabook.org/en/posa/chrome-images/multiproc.png "图1.3, 多进程架构")

### 进程间通信和多进程资源加载

进程间通信略

![alt text](http://www.aosabook.org/en/posa/chrome-images/ipc.png "图1.4 进程间通信")

这种架构的好处是所有资源请求都完全放在I/O线程处理, UI行为或者网络事件都不会互相影响. 资源过滤在浏览器进程的I/O线程中运行, 处理资源请求消息, 并发送给浏览器进程的单例ResourceDispatcherHost. 

这个单例接口使得浏览器可以控制每个渲染器对网络的访问, 也提供了有效持续的资源共享. 例如: 

 * **套接字池和连接限制:** 浏览器可以强制限制每个profile(256), 每个proxy(32)和每个`{scheme, host, port}`组的套接字数目. 注意对相同的`{host, port}`, 可以有6个HTTP和6个HTTPS连接. 

 * **套接字复用:** 永久TCP连接在服务请求之后会在套接字池里保留一段时间用以实现连接复用, 这样可以避免新的连接有额外的DNS, TcP和SSL(需要的话)设置开销.  
 
 * **Socket late-binding:** 只有在套接字准备分发应用请求时, 请求才和底层TCP连接关联在一起, 这样可以更好地处理请求优先级(例如在套接字建立连接时来了一个优先级更高的请求), 达到更高的吞吐量(例如, 在打开新连接的时候一个套接字空闲下来, 这时候可以重用"活跃的"TCP连接), 也可以作为TCP预连接的常规机制或者其他几个优化. 
 
 * **Consistent session state:** 所有渲染进程可以共享认证, cookie, 和缓存数据. (疑问, 既然不同渲染进程表示不同网站, 共享认证, cookie, 缓存有什么用呢?)
 
 * **预测优化:** 通过观察网络传输流量, chrome可以建立并完善预测模型来改进性能
 
渲染进程需要关心的只是简单地通过IPC发送资源请求消息给浏览器进程, 这个消息有一个唯一ID, 浏览器进程会处理其他的. 

### 跨平台资源获取

Chrome的network stack实现的一个主要问题是多平台的移植: Linux, Windows, OS X, Chrome OS, Android和iOS. 为了解决这个问题, network stack主要是作为单线程跨平台库来实现的(有几个别的), 这样Chrome可就可以重用同样的结构并提供同样的性能优化, 从而更有可能在不同平台上优化. (OO..)

代码在`src/net`里. 目录结构不言自明. 表1.2是几个例子. 

表1.2 - chrome的组件

|Component|Description|
|:---------:|:---------:|
|net/android	|Bindings to the Android runtime
|net/base	|Common net utilities, such as host resolution, cookies, network change detection, and SSL certificate management
|net/cookies	|Implementation of storage, management, and retrieval of HTTP cookies
|net/disk_cache	|Disk and memory cache implementation for web resources
|net/dns	|Implementation of an asynchronous DNS resolver
|net/http	|HTTP protocol implementation
|net/proxy	|Proxy (SOCKS and HTTP) configuration, resolution, script fetching, etc.
|net/socket	|Cross-platform implementations of TCP sockets, SSL streams, and socket pools
|net/spdy	|SPDY protocol implementation
|net/url_request	|URLRequest, URLRequestContext, and URLRequestJob implementations
|net/websockets	|WebSockets protocol implementation

代码写的很好, 慢慢读. 测试用例也很多. 

### 移动平台架构 (略)

### Chrome的Predictor的投机式优化

浏览器进程有个`Predictor`单例, 会观察网络模式来学习预测用户行为. 例如: 

 * 鼠标hover的时候偷偷建立TCP连接
 * URL输入时有建议网址, 可以建立DNS查询, TCP预连什么的. 
 * 收藏夹预连
 
Chrome学习网络拓扑结构的同时也学习你的上网模式. 就像一个有眼色的小厮一样预测你的行为事先做好准备带来"立即响应"的体验. Chrome为此投资了4个核心优化技术: 

表1.3 - Chrome用到的网络优化技术

|技术|描述|
|:---------:|:---------:|
|DNS pre-resolve	|提前解析主机名
|TCP pre-connect	|提前连接服务器
|Resource prefetching	|提前获取关键资源, 加速渲染
|Page prerendering	|提前获取整个页面

然后讲了一点具体应用`Predictor`的代码, 先略过. 

提到`connectInterceptor`对象被夹在请求里, 用来观察网络数据. 

用户的每一点小动作, 渲染器进程都有可能发送给浏览器, 做预热什么的. 

### Chrome 网络结构

 * 多进程, 浏览器进程, 渲染进程
 * chrome持有一个单例的资源分配器, `ResourceDispatcherHost`, 所有渲染进程共享, 在浏览器进程运行. 
 * network stack是跨平台, 多数情况下单进程的库
 * network stack不做阻塞操作
 * 共享network stack的种种好处
 * 渲染进程通过IPC和资源分配器交互
 * 资源分配器通过IPC filter拦截一些资源请求
 * Predictor 拦截资源请求和传输交通学习和优化网络请求
 * Predictor 可能会(看上面那个表)安排DNS, TCP, 甚至资源请求啥的. 

## 浏览器session的生命周期

每个阶段的优化. 不确定先跳过的话能否看的懂代码. 
