WebContents类对应于一页面窗口
创建WebContents需要填充WebContents::CreateParam参数。在CreateParam参数中需要指定: BrowserContext(使用net::URLRequestContextGetter获取URL请求所需要的URLRequestContext结构和资源加载时需要的数据)对象、SiteInstance对象、routing_id(用于路由消息的发送目的地、initial_size指定初始大小、context暂时没用上

// WebContents is the core class in content/. A WebContents renders web content
// (usually HTML) in a rectangular area.
//
// Instantiating one is simple:
//   scoped_ptr<content::WebContents> web_contents(
//       content::WebContents::Create(
//           content::WebContents::CreateParams(browser_context)));
//   gfx::NativeView view = web_contents->GetView()->GetNativeView();
//   // |view| is an HWND, NSView*, GtkWidget*, etc.; insert it into the view
//   // hierarchy wherever it needs to go.
//
// That's it; go to your kitchen, grab a scone, and chill. WebContents will do
// all the multi-process stuff behind the scenes. More details are at
// http://www.chromium.org/developers/design-documents/multi-process-architecture .
//
// Each WebContents has exactly one NavigationController; each
// NavigationController belongs to one WebContents. The NavigationController can
// be obtained from GetController(), and is used to load URLs into the
// WebContents, navigate it backwards/forwards, etc. See navigation_controller.h
// for more details.

//WebContents是核心类. WebContents在一块方形区域中渲染web content. 
// 初始化
// WebContents有一个NavigationController, 1<->1

SiteInstance在BrowsingInstance注册备案, 
BrowsingInstance(理论上)与BorwsingEcontext一一对应
BrowsingInstance内部维护一个站点URL(如:baidu.com)到SiteInstance对象的映射，并提供注册/取消注册、查找的接口。相当于SiteInstance的cache对象。

BrowsingInstance对象为诸多SiteInstance对象所共享,同样采用引用计数的方式控制生命周期

NavigationController接口：负责页面的导航逻辑：前进、后退、刷新等。其内部维护一个导航过的NavigationEntry列表

使用cef时, CefFrame作为UI的接口加载url, 最终调用CefBrowserHostImpl(同时也是content::WebContentDelegate的一个实现)::LoadURL(). 对URL做一些矫正补充, 

发给web_contents_->GetController()->LoadURL(), web_contents->getController()是navigation_cotnroller, 介个controller会执行LoadURL的主要工作(我猜的...), 这个controller有一个BrowserContext, 什么时候设置给controller还没找到. 

如果不是已经访问过的entry, web_contents_会怎样跳转过去?

这前面做了好多事情, 都在浏览器进程里, 最终浏览器进程调用WebContentsImpl::NavigateToEntry >> content::RenderViewHostImpl::Navigate(), 后者从浏览器进程向渲染进程发送一个"Send(new ViewMsg_Navigate(GetRoutingID(), params));"的消息. 

---------------------浏览器进程结束, 进入渲染进程 --------------------------
RenderViewImpl把对这类消息的处理放在RenderViewImpl::OnNavigate中. 这时已经在渲染进程里了. 
在RenderViewImpl中, 我们要
  // We will need to send the request's httpBody data up to the
  // browser process, and issue a special POST navigation in WebKit (via
  // FrameLoader::loadFrameRequest).
在::OnNavigate()中到l:1280, RenderViewImpl做到了 FrameLoader::loadFrameRequest

这是WebKit内部调啊调到这里了, 实在跟不下去了. 
webcore::ResourceHandleInternal::start()
webkit_glue::WebURLLoaderImpl(WebKit::WebURLLoader的实现)::loadAsynchronously
那么WebKit内部又是怎么跑出来调用渲染进程的IPC发消息给浏览器呢? 

通过在webkit_glue实现纯虚接口WebURLLoader做的, WebKit被命令加载URL时最终会调用WebURLLoader来异步加载(多数为异步加载), 使用者实现WebURLLoader接口来定制加载实现, webkit_glue的实现, 就会发个消息给IPC. 

IPCResourceLoaderBridge::Start(peer)
IPCSender::Send(
      new ResourceHostMsg_RequestResource(routing_id_, request_id_, request_)) //resource_dispatcher.cc l:185
	  
---------------------我是渲染进程和浏览器进程的分割线----------------------

又回到浏览器进程, 很重要的浏览器进程单例ResourceDispatcherHostImpl::OnRequestResource被调用了
ResourceDispatcherHostImpl::OnMessageReceived > ResourceHostMsg_RequestResource > 
ResourceDispatcherHostImpl::BeginRequest (貌似, 同步异步都会用这个)
	这里面会调用content::ResourceLoader, 从deferred_loader里面找一个(找的方式没看懂)
	一堆过滤, 该不该block, 有没有观察者想要block之后
	ResourceLoader 是Chromium浏览器实际的资源加载类，它负责管理请求到网络，从网络过来的认证请求，回复的管理等工作。因为这其中每项都有专门的类来负责，但是都是有ResourceLoader来管理这些。
	
	终于创建net::URLRequest了 (~~o(>_<)o ~~ 跟了n天了)
	URLRequest类从网络或者本地文件读取信息，它实际上承当了建立网络连接，发送请求数据和接受回复数据的任务. 在URLRequest之下就是Chromium的网络模块部分. 
	
		deferred_loader里有的话就直接用
		没有的话就从net::URLRequestContext里搞一个出来
		把这个request一顿捯饬
		scheme判断一下什么的.
		设置给ResourceHandler (ResourceLoader使用SyncResourceHandle和AsyncResourceHandle来向Renderer进程发送状态消息，并接受Renderer进程对这些消息的反馈。)
		content::ResourceDispatcherHostDelegate *delegate_->RequestBeginning (没做什么?)
		
		就是前面那个deferred_loader ResourceLoader::CompleteTransfer 
		最终调到URLRequest::Start()
		URLRequest::StartJob(URLRequestJobManager::GetInstance()->CreateJob(
      this, network_delegate_)); (见注[1])
	  
			注[1]: URLRequestJob的获取: 对于URLRequestJob和它的工厂URLRequestJobFactory的管理工作都是URLRequestJobManager负责。基本的思路是，用户可以在该类中注册多个工厂，当有URLRequest请求时候，先有工厂检查它是否需要处理该scheme，如果没有，继续交由下一个工厂类。最后，如果没有任何工厂能够处理的话，则交给内置的工厂来检查和处理是否是http://，ftp://或者file://等。
			
		调用到URLRequestHttpJob::Start()
		//做一些处理
		其次，当URLRequestHttpJob被创建后，它首先从Cookie管理器中获取跟该URL相关联的信息。之后，它同样借助于HttpTransactionFactory创建一个HttpTransaction类的对象来表示开启一个HTTP连接的事务（当然这里的概念不同于数据库中的事务概念）。通常情况下，HttpTransactionFactory对应的是一个它的子类HttpCache的实例。HttpCache类使用本地磁盘缓存机制（稍后会介绍），如果该请求对应的回复已经在磁盘缓存中，那么无需再建立HttpTransaction来发起连接，直接从磁盘中获取即可。如果磁盘中没有，同时如果目前该URL请求对应的HttpTransaction已经建立，那么只要等待它的回复即可。这些条件都不满足后，实际上才会真正创建HttpTransaction。
		
		URLRequestHttpJob::AddCookieHeaderAndStart()
		URLRequestHttpJob::CheckCookiePolicyAndLoad()
		等cookie彻底load完了
		URLRequestHttpJob::DoStartTransaction()
		URLRequestHttpJob::StartTransactionInternal()
			在这里借助于HttpTransactionFactory创建HttpTransaction
		transaction_->Start(
            &request_info_, start_callback_, request_->net_log())
		同时告诉URLRequestDelegate
		MessageLoop::current()->PostTask(
      FROM_HERE,
      base::Bind(&URLRequestHttpJob::OnStartCompleted,
                 weak_factory_.GetWeakPtr(), rv));
				 
		(>>3)HttpTransaction::Start()
			有 HttpNetworkTransaction : public HttpTransaction 
			或 HttpCache::Transaction : public HttpTransaction
		//再次，HttpNetworkTransaction使用HttpNetworkSession来管理连接会话。HttpNetworkSession通过它的成员HttpStreamFactory来建立TCP Socket连接，之后创建HttpStream对象。HttpStreamFactory将和网络之间的数据读写交给自己新创建的一个HttpStream（其实是它的子类）对象来处理。
		
		//HTTPNetworkTransaction 不光是HttpTransaction的实现, 也是HttpStreamRequest::Delegate 的一个实现, 后者是一组回调接口, 用来在stream状态发生变化的时候被调用

		//HttpNetworkTransaction内部采用状态机标记transaction state, 调用HttpNetworkTransaction::Start时会把状态置为"STATE_CREATE_STREAM", 然后开始跑状态机, HttpNetworkTransaction::DoLoop(), 创建Stream(HttpNetworkTransaction::DoCreateStream)使用net::HttpNetworkSession, net::HttpNetworkSession使用工厂创建Tcp socket连接
		
			(>>2)HttpNetworkTransaction::DoCreateStream
			(>>1)HttpStreamFactoryImpl::RequestStream
			
			开始在 HttpStreamFactoryImpl::Job::DoLoop 里绕啊绕
			这里面经历了解析代理, 创建连接, 创建stream几个过程
			
				STATE_START 什么都没做 
				HttpStreamFactoryImpl::Job::DoStart() //还没看见创建tcp socket连接(~~o.o~~)
				
				STATE_RESOLVE_PROXY 
				HttpStreamFactoryImpl::Job::DoResolveProxy()
				session_->proxy_service()->ResolveProxy(
			  request_info_.url, &proxy_info_, io_callback_, &pac_request_, net_log_)
				
				STATE_RESOLVE_PROXY_COMPLETE
				DoResolveProxyComplete()
				
				假设为non blocking job, 因为发送请求, 不要求马上返回结果
				
				STATE_INIT_CONNECTION
				HttpStreamFactoryImpl::Job::DoInitConnection()
					//不一定创建连接, 没有可用spdy session, 也没有preconnecting的时候才会调用这个
					// 创建连接的方法, 使用传入的proxy info初始化一个ClientSocketHandle, 这个方法用于Http/Https请求, 方法声明在
					src/net/socket/client_socket_pool_manager.h l:92
					此处我们先只关注**|resolution_callback|**, 回调方法先不关注, 没有关键作用
				
				上面这个方法会跟到这里: 这个方法的返回值后面还会用到, 所以我跟了一下
				int InitSocketPoolHelper(...) client_socket_manager.cc l:69
				这里面是具体的, 解析ssl的, 创建tcp socket的操作, 加上代理(有的话), 
				使用ssl有一种处理, http/https有一种处理, socks有一种处理, 直连一种处理, 每一种都分为有预连接(返回OK)和没有预连接要初始化连接(返回初始化结果)两种情况.
				
				初始化连接时会返回 pool->RequestSocket() 的结果
				
				//注[2]http://www.chromium.org/developers/design-documents/network-stack#TOC-Anatomy-of-a-Network-Request-focused-on-HTTP- 在HostResolution下面讲
				transport sockets的connection setup 不光要求TCP握手, 也要求host resolution. HostResolverImpl使用 getaddrinfo() 执行host resolution, 这是个 blocking call, 所以 resolver在其他的工作线程里调用这些方法. host resolution 通常也会掺合DNS解析... 注意, 在写的时候, 同时进行的host resolution不超过8 (还在改善优化). 
				但是, 在STATE_INIT_CONNECTION 和 STATE_INIT_CONNECTION_COMPLETE 两个阶段我都没找到哪里做的TCP握手...
				
				//注[3]
				但是看下面一段
				SSL
				SSL socket需要进行SSL 连接配置和认证验证. 现在在所有平台下我们都在用NSS的libssl库处理ssl连接逻辑. 但是认证验证用的是平台相关的API. 我们正在朝着使用认证验证缓存的方向努力, 这样就可以把同一个认证的多次验证整合成一个认证验证任务, 在缓存里放一会儿. 
				
				`SSLClientSocketNSS`大致地做这些事情(忽略几个高级特性)
				* Connect()调用. 我们在`SSLConfig`确立的配置或预编译宏的基础上设置NSS的SSL选项. 然后启动握手. 
				* 握手结束了. 假设没有遇到错误, 继续使用`CertVerifier`验证服务器的认证. 认证验证会花一阵子, `CertVerifier`使用`WorkerPool`来真正调用`X509Certificate::Verify()`, 平台相关API会有具体实现. 
				
				
				STATE_INIT_CONNECTION_COMPLETE
				HttpStreamFactoryImpl::Job::DoInitConnectionComplete() 没看懂
				
				STATE_CREATE_STREAM
				HttpStreamFactoryImpl::Job::DoCreateStream
				
				STATE_CREATE_STREAM_COMPLETE
				HttpStreamFactoryImpl::Job::DoCreateStreamComplete
				
				STATE_NONE
				完了
			(<<1)
			(<<2)
		
		(还在>>3里, 还在HttpNetworkTransaction::DoLoop()里...)
			STATE_CREATE_STREAM_COMPLETE
			HttpNetworkTransaction::DoCreateStreamComplete()
			有错误的话处理错误
			
			STATE_INIT_STREAM
			HttpNetworkTransaction::DoInitStream()
			
			STATE_INIT_STREAM_COMPLETE
			HttpNetworkTransaction::DoInitStreamComplete
			
			STATE_GENERATE_PROXY_AUTH_TOKEN
			HttpNetworkTransaction::DoGenerateProxyAuthToken
			
			STATE_GENERATE_PROXY_AUTH_TOKEN_COMPLETE
			HttpNetworkTransaction::DoGenerateProxyAuthTokenComplete
			
			STATE_GENERATE_SERVER_AUTH_TOKEN
			HttpNetworkTransaction::DoGenerateServerAuthToken
			
			STATE_GENERATE_SERVER_AUTH_TOKEN_COMPLETE
			HttpNetworkTransaction::DoGenerateServerAuthTokenComplete
			
			STATE_INIT_REQUEST_BODY
			HttpNetworkTransaction::DoInitRequestBody
			
			STATE_INIT_REQUEST_BODY_COMPLETE
			HttpNetworkTransaction::DoInitRequestBodyComplete
			
			STATE_BUILD_REQUEST
			HttpNetworkTransaction::DoBuildRequest
			
			STATE_BUILD_REQUEST_COMPLETE
			HttpNetworkTransaction::DoBuildRequestComplete
			
			STATE_SEND_REQUEST
			HttpNetworkTransaction::DoSendRequest
			stream_->SendRequest(request_headers_, &response_, io_callback_);
			(疑问: hostname解析的回调方法还没有被调用的话, 这里怎么就可以发送请求了?)
			(跟进去又是一个DoLoop(), 没有看..., 崩溃)
			
			STATE_SEND_REQUEST_COMPLETE
			HttpNetworkTransaction::DoSendRequestComplete
			
			STATE_READ_HEADERS
			HttpNetworkTransaction::DoReadHeaders (stream_->ReadResponseHeaders)
			
			STATE_READ_HEADERS_COMPLETE
			HttpNetworkTransaction::DoReadHeadersComplete
			这里没有继续设置next_state_的状态, 除了维持 STATE_READ_HEADERS ...
			
			