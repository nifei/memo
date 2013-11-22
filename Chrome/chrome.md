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
这个类持有浏览器对话所需的上下文. 它在UI线程中. 所有的方法只能在UI线程中调用. 
