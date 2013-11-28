Chrome 的类关系:

* ProfileIOData
Profile持有的IO线程中的数据, 包括但不限于network object (CookieMonster, HttpTransactionFactory ==). Profile持有ProfileIOData, 但是会保证在IO线程中析构它. 

* ProfileImplIOData : public ProfileIOData

* OffTheRecordProfileIOData : public ProfileIOData
OffTheRecordProfile持有OffTheRecordProfileIOData::Handle, handle引用OffTheRecordProfileIOData. 这个类旨在持有OffTheRecordProfile在IO线程持有的所有对象, 包括但不限于network objects(CookieMonster, HttpTransactionFactory ==). OffTheRecordProfileIOData被OffTheRecordProfile和OffTheRecordProfileData的ChromeURLRequestContexts持有. 它们都被删掉了, OffTheRecordProfileIOData就会被析构. 注意OffTheRecordProfileIOData通常比持有它的Profile活的更久, 所以除了LazyParams使用的Profile之外, OffTheRecordProfileIOData不可以引用Profile. 

* Profile : public content::BrowserContext
其实这篇官网的文档不是很好. 感觉那个调用关系的图以后追踪bug的时候有用的样子. http://dev.chromium.org/developers/design-documents/profile-architecture
Profile服务通过下面的参数被使用. 这个参数定义了调用者想要用Profile服务干什么. 
今后, 我们还可以用Profile来做进一步的检查. 比如说返回一个只允许部分方法的History service. 

* content::BrowserContext


TODO:
流程图要补充:
Create WebContents --> CreateRequestContext
	BrowserContext在Cef中时PreMainMessageLoop时显示创建的. 在这里Cef还会调用一下BrowserContext::GetRequestContext 初始化RequestContext
	在content shell也是在PreMainMessageLoop创建BrowserContext, 但是它持有的RequestContext要到创建WebContent(开启第一个窗口)时才会创建. 但是content shell在这里就会创建一个新窗口, 所以requestContextGetter也在message loop运行之前就创建了. 
	chrome怎么实现的还没有看, 打开这个solution太久了...Mark一下回头看.

track BrowserContext需要注意的几个问题: 
1. protocol handler在传入之前都读入了哪些设置?
	StoragePartitionImplMap 里会塞进去一些, 这个类的实现
	StoragePartitionMap什么时候初始化的.
	StoragePartition会用到很多
	shellurlrequestcontextgetter持有protocol_handler和network_delegate
	ShellUrlRequestContextGetter为什么需要持有两个broswer的loop?
	为什么RequestContext要区分普通的和storage partition的?
	(IOThread)ShellURLRequestContextGetter创建URLRequestContext的时候会assign给它(在原有的基础上, 因为ShellURLRequestContextGetter被创建的时候已经有protocol_hanlder了)一些新的protocol_handler(这些会影响scheme的解析); 还会给它自己的network_delegate, which会处理URLRequest运转周期的一系列事件.还会给它自己掌控的CookieStore. 
	
	URLRequestContext这个持有protocol_handler, cookie_store和network_delegate(目前为止只关心这三个)的类在创建urlRequest的时候给了urlRequest访问自己这些成员变量的权限, 所以urlRequest在被URLRequestJob/HttpJob/HttpTransaction运转的时候可以传递这些玩意.
	
	protocol_handler 可以用来解析scheme, 创建不同的URLRequestJob
	network_delegate前面说过了
	cookie_monster 读写cookie
	
	HttpJob->Start之后会调用context->cookie_store()读取cookies.
	什么时候写入cookiestore的, cookiestore的初始化问题:
	
	问题A: 浏览过程中产生的cookie是什么时候写入cookie_store的;
	两种: 一种是解析js时发现js设置了一些cookie, 这时候RenderMessageFilter会收到OnSetCookie的消息(因为是RenderProcess中的V8解析js做的动作, 所以消息时从渲染进程通过IPC发到浏览器进程的)(看下登录豆瓣的几个cookie除了用户名和密码, 那是豆瓣使用google的一个服务(问下服务名字)在页面上嵌入js插入的cookie用来统计用户浏览行为的);
	一种是httpjob->start之后的OnStartComplete时会调用下HttpJob::SaveCookiesAnd什么的. 这时候ResponseHeader已经拿到了, 里面会有cookie(例如登录)
	
	问题B: 初始化cookie monsters(这个类是不是一开始实现的时候很可怕?)
	cookie_monster的创建 urlrequestcontextgetter一般会顺手创建它(谁管生命周期?), 并且给它两个cookiable schemes (http/https)
	
	continue.. CefURLRequestContextGetter::GetURLRequestContext
	初始化CookieMonster时会传一个SQLitePersistentCookieStore给它.
	
	SqllitePersisitentCookieStore加载数据库不是在最开始, 而是在需要的时候才去load
	load的任务在IO线程由URLRequest(例如,HttpJob)引发, CookieMonster调用SqllitePersistentCookieStore去load, 但是任务要放到DB thread去执行, 所以load(url, cookie)这件事情传来的时候会带一个回调方法, 加载完了调用. 
	
	SQLitePersistentCookieStore::LoadAndNotifyInBackground 时如果没有数据库文件, 会创建一个. 
	
	浏览过程中产生的cookie如果需要保存下来,会在浏览器退出的时候flush到数据库. 
	
	CookieMonster和SqlLitePersistentCookieStore的实现细节content的使用者可以不关心, 直接创建拿来用就好了. 
	
	
从流程图"URLRequest简化版"我们可以看到:
	发生在浏览器进程UI线程的蓝色部分:
	content::WebContents::Create() 过程: 
	在打开一个新窗口(可以先认为一个窗口一个渲染进程)时会通过定制的CefContentBrowserClient创建CefBrowserContext, 这个类持有浏览器对话所需的上下文. 它在UI线程中创建, 所有的方法只能在UI线程中调用; 
CefContentBrowserClient会在适当的时候通过CefBrowserContext::CreateRequestContext创建UrlRequestContextGetter, 这个类负责创建URLRequestContext, 包含URL请求所需的上下文. 这个调用需要被保证发生在UI线程. 
	返回的定制的CefURLRequestContextGetter会在IO线程中被调用创建URLRequestContext, 后者在IO线程URLRequest的创建, 准备, 发送, 回应中起到重要的作用. 它不直接提供信息或者代理, 但是它"持有"(因为其实是通过URLRequestContextStorage持有)代理或者处理者供UrlRequest使用. 
	
	content::WebContents::LoadURL()过程:
	这个调用会在窗口接收一个新的URL时发生. 在给定制的CefContentBrowserClient一些机会处理, 并且问它要了一些资源, 询问一些属性之后, 通过IPC向渲染进程发消息加载这个URL. 

	发生在绿色的渲染进程的部分:
	渲染进程做了什么我不知道, 反正它发给浏览器进程消息, 要求它请求资源; 
	
	粉色的两根线表示进程间的IPC通信. 
	
	剩下的就是发生在浏览器进程的IO线程的黑色部分:
	
	IO线程的ResourceDispatcherHost收到消息以后, 先通过URLRequestContext创建一个URLRequest(怎么创建的, 因为URLRequestContext是由定制的CefURLRequestContextGetter创建的, 所以这里创建URlRequest的参数可以定制);
	然后ResourceDispatcherHost会把UrlRequest交给ResourceLoader来运转. 主要流程便是开始Request, 创建RequestJob, 在response准备好的时候读数据, 读完了做一些释放动作. ResourceLoader不可以定制, 所以这个流程我们不能改变. 但是几乎每一步动作ResourceLoader或者URLRequest/URLRequestJob都会访问URLRequestContext带有的"network_delegate", "protocol_handler", "cookie_store"和它自身, 或是询问权限, 或是直接使用处理者. 
	
	所以在另一张流程图"URLRequest的cookie store和protocol handler"里我们重点看一下这几个东西的使用. 这也是我们定制scheme, 处理cookie的切入点. 









