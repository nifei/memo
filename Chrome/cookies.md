Cookies 在HTTP权威指南中要注意的事情. 

两种cookie, 0 & 1 在11.6.6和11.6.7 字段, 混用, 什么的. 

11.6.9 cookie and caching

反正response header里有set cookie时注意一下 11.6.9

关于CookieMonster, 官网文档已经讲的很详细了, 

有关作为cookie键值的几个术语: 
TLD: top level domain, "."后面的, "com" 之于 "google.com"或"world.std.com"
eTLD: effective TLD, "."后面的, 带有通用域名注册的最长的域名, "co.uk"之于"news.bbc.co.uk"
eTLD+1: 不受通用域名注册控制的, 用"."分开的后半部分的字符串. ("bbc.co.uk" 之于 "news.bbc.co.uk", 或 "google.com"之于 "news.google.com")

它们都应该带上一个".", 数据库里存的都带"." o(╯□╰)o, 注释里讲的时候也都说".intiated". 

作为content api的使用者, 我们掌握CookieMonster这个类的用法就够了. 这个类详细的使用方法见http://www.chromium.org/developers/design-documents/network-stack/cookiemonster

CookieMonster的获取/设置
在net模块内部使用net::URLRequestContext::set_cookie_store()/cookie_store() 来设置和获取cookie_store. 

### 关于 URLRequestContextGetter

net::URLRequestContextGetter被谁调用提供一个 net::URLRequestContext* GetURLRequestContext() 的实现返回定制的net::URLRequestContext. 

content shell和cef直接创建一个urlrequestcontext然后做一些设置返回. 
chrome通过定制工厂类URLRequestContextFactory返回不同的urlrequestcontext子类. 

content shell和cef对URLRequestContextGetter的定制比较简单, 都为URLRequestContext创建或指定了一个storage, 一个network delegate, 创建Http transaction 的工厂(net::HttpCache), 和控制生成URLRequestJob逻辑的protocol handler. 二者都有直接/间接调用urlrequestcontext::set_cookie_store(new cookieMonster)

(storage对业务逻辑并无影响, 是为了take ownership of urlrequestcontext memebers 保证不引用野指针而存在的, 使用者要么在urlrequestcontext的子类接管成员, 要么创建这么一个storage作为存储). 

对业务逻辑有影响的是cookie store, protocol handler 和 http transaction factory. 


* 关于 net::HttpCache
net::HttpCache 提供了一种可以添加Http 缓存的可叠加的 net::HttpTransactionFactory的实现, 缓存逻辑遵循RFC 2616. net::HttpCache 接受一个 disk_cache::Backend 参数用来做缓存存储媒介(AppData/cache之类的).

Http cache就是http/1.*为了尽量减少重复的内容的传输而设计的一些元素.
http://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html
 