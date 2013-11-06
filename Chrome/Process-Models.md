# Process Models

这篇文档描述了Chomium支持的不同进程模型和现存的警告. 

## Overview

> 网页内容现在先进死了. 浏览器更像小型操作系统而不是简单的文档渲染器. Chromium就致力于像操作系统一样又安全又健壮地运行这些应用, 使用多系统进程来隔离网站之间也隔离浏览器本身. 这样做会更健壮是因为每个进程在自己的地址空间上运行, 由操作系统来调度, 挂了也只挂自己一个. 用户可以在资源管理器中查看Chromium的每个进程资源用度(怪不得又快又占资源)

> 浏览器有很多种方法可以划分到不同的系统进程中, 综合考虑很多因素包括稳定性资源管理和实际应用的观察才能选择最好的一种. Chromium支持四种进程模型供体验, 默认的那个比较适合大多数用户. 

## Supported Models

Chromium支持四种进程模式, 影响浏览器分配页面给渲染进程的方式. 默认情况下Chromium为用户访问的每个网站示例分配一个单独的系统进程. 用户可以通过指定启动Chromium的命令行来使用其他的架构: 一个网站的所有实例使用一个进程, 每组连在一起的tab使用一个进程, 或者每个东西使用一个进程. 这些模式不同之处在于是否反应原始内容, tab之间的关系. 本小节详细讨论每种模式; 然后会讲Chromium现在的实现需要注意的地方. 

### Process-per-site-instance

默认模式. 来自不同网站的内容都会独立渲染, 对相同网站的访问也会彼此隔离. 一个网站的崩溃或者超载不会影响到浏览器. 这种模式基于原生内容和相互描述的tab之间关系. 

两个tab可能显示在同一个进程中渲染的页面(不懂), 跨网站的页面跳转会切换到另一个tab的渲染进程. (这里有很重要的注意事项!!在Caveates小节有讲). 

具体讲, 我们定义一个"Site"是一个注册的域名(google.com, bbc.co.uk)加上一个scheme(https://). 这和Same Origin Policy定义的origin差不多, 但是子域名(mail.google.com, docs.google.com)和端口(http://foo.com:8080)被分组在同一网站下. 这样做是为了让来自同一网站不同子域名或者 不同端口的页面可以互相通过Javascript互相访问 - 如果socument.domain变量相同, 这样做是Same Origin Policy允许的. 

一个"网站实例"是相互联系的同一网站的页面的集合. 两个页面是相连的意思是它们可以在脚本里互相访问(一个页面使用Javascript打开另一个). 

#### 优点

* *来自不同网站的分离内容*. 这对网页内容的致命分享有重大意义, 页面与其他网站导致的崩溃分离开来. 

* *显示同一网站的独立tab的分离*. 在不同tab里访问相同网站会创建不同的进程. 这样避免了一个实例对其他实例的影响. 

#### 缺点

* *内存开销*. 介个模式比下面process-per-site那个模式产生更多的进程. 增进稳定性和并行机会的同时也增加了内存开销. 

* *更复杂的实现*. 和process-per-tab和single-process不同, 这个模式需要复杂的逻辑来支持tab切换时的进程切换, 以及托管一小部分origins之间允许的Javascript行为, 例如postMessage. (这个问题的更多探讨以及我们为此做的努力请见下面注意事项部分和[Site Isolation](http://www.chromium.org/developers/design-documents/site-isolation)项目页面). 

### Process-Per-site

Chromium也支持把不同网站隔离开, 但是把同一网站的所有实例放在同一个进程的模式. 用户需在启动Chromium时再命令行中指定`--process-per-site`. 这种模式渲染进程更少, 用健壮性换取内存开销. 这个模式基于"origin of the content and not the relationships between tabs". 

#### 优点

* 略过, 以后再说

### process-per-tab

process-per-site-isntance和process-per-site模式在创建进程时都考虑内容的源点. Chromium还支持一种更简单的模式, 给每组脚本相连的tab一个进程. 这种模式需要通过`--process-per-tab`指定. 

具体讲, 我们把通过脚本连接在一起的一组tab看做一个*浏览实例*, 和Html5中"(unit of related browsing contexts)相关浏览context单元"概念相对应. 这个集合由一个tab和这个tab通过javascript打开的tab组成. 这样的tab必需在同一进程中渲染这样它们之间的javascript调用才能用(通常发生在同源页面间). 

#### 优缺点跳过

### Single process

作为比照Chromium支持单进程模式, `--single-process`可以指定这种模式. 在这种模式下浏览器和渲染引擎在一个进程中运行. 

单进程模式为多进程架构的开销提供了比较的基准. 这个模式既不健壮也不安全, 任何渲染器的崩坏都会终止整个浏览器进程. 这是为了测试和开发而设计的, 可能存在其他模式不存在的问题. 

## Sandboxes and plug-ins

每种多进程架构中, Chromium渲染进程都在限制访问的沙盒进程中运行. 这些进程没有权限直接访问用户的文件系统, 输出和其他资源. 它们只能通过浏览器进程访问资源, 这样可以强加安全措施. 这样Chromium的浏览器进程可以把损害减轻到渲染引擎可以处理的程度. 

浏览器plug-in像flash和silverlight也可以在各自的进程中运行, 像flash甚至可以运行在chromium的沙盒中. 在chromium支持的每个多线程架构中, 都为每个类型的plug-in分配一个进程实例. 这样所有的flash实例运行在一个进程中, 不管它们在哪个网站或页面里显示

## Caveats

这一块列出了Chromium现在实现的注意事项以及牵涉. (**这块貌似比较重要, 前面也说了**)

* 多数渲染器开启的页面内跳转不会切换进程. 用户点一个链接, 提交一个表, 或脚本重定向, 如果跳转跨网站的Chromium不会切换tab里的进程. Chromium只会对浏览器开启的跨网站跳转切换渲染进程, 比如在地址栏里输入URL或者点击书签. 这样的话来自不同网站的页面有可能在同一进程中渲染, 即使在process-per-site-instance和process-per-site-models也是这样. 这一点作为[Site Isolation](http://www.chromium.org/developers/design-documents/site-isolation)项目的一部分可能在Chromium今后的版本中改了. 

但是网页可以使用一种机制建议指向无关页面的链接, 在不同进程中安全地渲染. 如果链接有`rel=noreferrer target=_blank`这样的属性chromium通常会在不同进程中渲染它. 

* 子版面和父页面在统一进程中渲染. 尽管跨网站的子版面和父页面之间没有脚本访问也可以安全地在独立进程中渲染, Chromium么有这么做. 和第一条类似, 这意味着不同网站的内容可能在同一进程中渲染. chromium以后的版本可能会改. 

* Chromium可以创建的进程数是有限的. 不然浏览器会拖垮系统. 限制数目和计算机内存成正比, 最多80个. 这样一个进程就可能用于多个网站. 这种重用现在是随机的, chromium今后可能会使用启发算法更智能地分配渲染进程(我倒觉得使用启发算法等于给人机会用超载页面来黑你...). 

## Implementation notes

chromium中有两个类代表了各种进程模式的抽像: `BrowsingInstance`和`SiteInstance`. 

`BrowsingInstance`类代表浏览器内脚本相连的一组tab, 即HTML5 spec中的"unit of related browsing contexts". 在process-per-tab模式下, 我们为每个`BrowsingInstance`创建一个渲染进程. 

`SiteInstance`类代表了来自同一网站的相连的页面. 它是`BrowsingInstance`里页面的子集, 要记住`BrowsingInstance`里对每个网站只有一个`SiteInstance`. 在process-per-site-instance模式里我们为每个`SiteInstance`创建一个进程. 为了实现process-per-site, 我们得保证所有来自同一网站的`SiteInstance`都在同一个进程中渲染. 

## Academic Papers

介个才不翻译呢. 

[Isolating Web Programs in Modern Browser Architectures](http://www.charlesreis.com/research/publications/eurosys-2009.pdf)
Charles Reis, Steven D. Gribble  (both authors at UW + Google)
Eurosys 2009. Nuremberg, Germany, April 2009.
Abstract:
Many of today's web sites contain substantial amounts of client-side code, and consequently, they act more like programs than simple documents. This creates robustness and performance challenges for web browsers. To give users a robust and responsive platform, the browser must identify program boundaries and provide isolation between them.

We provide three contributions in this paper. First, we present abstractions of web programs and program instances, and we show that these abstractions clarify how browser components interact and how appropriate program boundaries can be identified. Second, we identify backwards compatibility tradeoffs that constrain how web content can be divided into programs without disrupting existing web sites. Third, we present a multi-process browser architecture that isolates these web program instances from each other, improving fault tolerance, resource management, and performance. We discuss how this architecture is implemented in Google Chrome, and we provide a quantitative performance evaluation examining its benefits and costs.

[Security Architecture of the Chromium Browser](http://crypto.stanford.edu/websec/chromium/)
Adam Barth, Collin Jackson, Charles Reis, and The Google Chrome Team
Stanford Technical Report, September 2008. 
Abstract:
Most current web browsers employ a monolithic architecture that combines "the user" and "the web" into a single protection domain. An attacker who exploits an arbitrary code execution vulnerability in such a browser can steal sensitive files or install malware. In this paper, we present the security architecture of Chromium, the open-source browser upon which Google Chrome is built. Chromium has two modules in separate protection domains: a browser kernel, which interacts with the operating system, and a rendering engine, which runs with restricted privileges in a sandbox. This architecture helps mitigate high-severity attacks without sacrificing compatibility with existing web sites. We define a threat model for browser exploits and evaluate how the architecture would have mitigated past vulnerabilities.
