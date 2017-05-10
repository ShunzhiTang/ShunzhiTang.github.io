---
layout: post
title: "基础篇-网络编程"
date: 2016-10-10 23:10:24.000000000 +09:00
tags: Objective-C回顾温习集
---
> 现在的APP大多数情况都是需要网络进行操作的，通过网络，一款应用才能够内容丰富，才能够完成用户操作和服务器后台数据数据的交互，网络编程是任何程序开发中不可缺少的部分，当然在iOS客户端也显得尤为重要。

### 一、网络编程
Cocoa中网络编程层次结构分为三层，自上而下分别是:
>
- Cocoa层: NSURL，Bonjour，Game Kit ，WebKit
- Core Foundation层： 基于C的CFNetwork和CFNetServices
- OS层： 基于C的BSD socket

注意： 在Xcode 7.0/iOS 9.0中苹果正式废弃了NSURLConnection系列的API，建议使用NSURLSession。因此下面的NSURLConnection的内容就当做复习参考。

**1、NSURLConnection** <br>
CoreFoundation 中提供了类NSURLConnection，用于处理用户的网络请求，NSURLConnection基本可以满足我们大多数的网络请求操作。NSURLConnection本身并不能单独使用，需要一些和通信有关的类进行协同工作。包括NSURLRequest，NSURLResponse，NSURLCache等等。<br>
工作原理：
>
&emsp;&emsp;NSURLRequest被传递给NSURLConnection。被委托对象（遵守以前非正式协议<NSURLConnectionDelegate>和<NSURLConnectionDelegate>）异步地返回一个NSURLResponse以及包含服务器返回信息的NSData。<br>
&emsp;&emsp;在一个请求被发送到服务器之前，系统会先查询共享的缓存信息，然后根据策略（policy）以及可用性（availability）的不同，一个已经被缓存的响应可能会被立即返回。如果没有缓存的响应可用，则这个请求将根据我们指定的策略来缓存它的响应以便将来的请求可以使用。<br>
&emsp;&emsp;在把请求发送给服务器的过程中，服务器可能会发出鉴权查询（authentication challenge），这可以由共享的cookie或机密存储（credential storage）来自动响应，或者由被委托对象来响应。发送中的请求也可以被注册的NSURLProtocol对象拦截，以便在必要的时候无缝地改变响应。发送中的请求也可以被注册的NSURLProtocol对象拦截，以便在必要的时候无缝地改变其加载行为。



***1.1 同步请求 ，使用sendAsynchronousRequest 方法*** <br>
&emsp;&emsp;这个同步请求是阻塞的，并且不可以中途cancel掉。我们可以将同步请求放在主线程之外的线程中，执行效果也类似于异步，比如放在GCD的dispatch_async里面执行。

***1.2 异步请求，使用sendAsynchronousRequest***<br>
&emsp;&emsp;这个异步请求是非阻塞的，异步执行后把结果通过block回调回来，中途不可以cancel。

***1.3 使用委托（delegate）请求*** <br>
&emsp;&emsp;使用NSURLConnectionDataDelegate协议进行网络请求的处理，具体查看一下文档，设置代理，重载代理方法，实现功能。这个异步请求是非阻塞的，异步执行后把返回的数据与结果通过delegate函数回调回来，可以使用cancel中途取消。

***1.4 将NSURLConnection的请求放在后台线程*** <br>
&emsp;&emsp;上面提到的NSURLConnection的异方式，实际上还是跑在主线程当中，在主线程中执行网络操作带来两个问题：
>
- 1.尽管在网络连接过程中不会对主线程造成阻塞，但是delegate的回调方法还是在主线程中执行。如果我们在回调方法中（特别是complete回调）中进行大量的耗时操作，仍然会造成主线程阻塞。
- 2.NSURLConnection默认会跑在当前的runloop中，并且跑在Default Mode ，当用户执行滚动UI操作时会发生runloop mode的切换，也就导致了NSURLConnection不能及时执行和回调。

解决这两个问题的答案在：[参考答案](https://hit-alibaba.github.io/interview/iOS/Cocoa-Touch/Network.html)


**2、NSURLSession** <br>
工作原理：
>
&emsp;&emsp;和NSURLConnection一样，NSURLSession指的不仅是同名类NSURLSession，还包括一系列相互关联的类。NSURLSession包括之前相同的组件，NSURLRequest与NSURLCache，但是把NSURLConnection替换成了NSURLSession、NSURLSessionConfiguration以及NSURLSessionTask的3个子类：NSURLSessionDataTask，NSURLSessionUploadTask，NSURLSessionDownloadTask。<br>
&emsp;&emsp;与NSURLConnection相比，NSURLSession最直接的改进就是可以配置每个session的缓存，cookie以及证书策略（credential policy），甚至跨程序共享这些信息。这将允许程序和网络基础框架之间相互独立，不会发生干扰。每个NSURLSession对象都由一个NSURLSessionConfiguration对象来进行初始化，后者指定刚才提到的这些策略以及一些用来增强移动设备上性能的新选项。<br>
&emsp;&emsp;与NSURLSession中另一大块就是session task，它负责处理数据的加载以及文件和数据在客户端以及服务端之间的上传和下载。NSURLSessionTask与NSURLConnection最大的相似支出在于他也负责数据的加载，最大的不同之处在于所有的task共享其创造者NSURLSession这一公共委托者（common delegate）。

***2.1 NSURLSessionTask*** <br>
NSURLSessionTask是一个抽象类，其下有3个实体子类可以直接使用：NSURLSessionDataTask , NSURLSessionUploadTask，NSURLSessionDownloadTask。这三个子类封装了现代程序三个最基本的网络任务：获取数据，比如JSON或者XML，上传文件和现在文件。

注意点：
>
- 当一个NSURLSessionDataTask完成时，它会带有相关的数据，而一个NSURLSessionDownloadTask任务完成时，它会带回已下载文件的一个临时文件路径。一般来说，服务器对于一个上传任务的响应也会有相关数据返回，所以NSURLSessionUploadTask继承自NSURLSessionDataTask.
- 所有的task都是可以取消，暂停或者恢复的。当一个download task取消时，可以通过选项来创建一个恢复数据（resume data），然后传递给下一次新创建的download task，以便继续以前的下载。
- 不同于直接使用alloc-init初始化方法，task是由一个NSURLSession创建的。每个task的构造方法都对应有或者没有completionHandler 这个block的两个版本。例如：有这样两个构造方法 ```-dataTaskWithRequest:``` 和```–dataTaskWithRequest:completionHandler:```这与NSURLConnection的```sendAsynchronousRequest:queue:completionHandler:```方法类似，通过指定completionHandler这个block将创建一个隐式的delegate。来替代该task原来的delegate -- session。对于需要override原有session task的delegate的默认行为的情况，我们需要使用这种不带completionHandler的版本。

使用NSURLSessionTask，就是按照官方文档去写就可以了。


***2.2 NSURLSession 与NSURLConnection的delegate方法*** <br>

具体观察：
>
- NSURLSession 即拥有session的delegate方法，有拥有task的delegate方法用来处理鉴权查询。session的delegate方法处理连接层的问题，诸如服务器信任，客户端证书的评估，NTLM和Kerberos协议这类问题，而task的delegate则处理以网络请求为基础的问题，如：Basic，Digest，以及代理身份验证（Proxy authentication）等。
- 在NSURLConnection中有两个delegate方法表明网络请求已经结束：```NSURLConnectionDataDelegate```中的```connectionDidFinishLoading:```和```NSURLConnectionDelegate```中的```-connection:didFailWithError:```,而在NSURLSession中改为一个delegate方法：```NSURLSessionTaskDelegate```的```-URLSession:task:didCompleteWithError:```。 NSURLSession中表示传输多少字节的参数类型现在改为int64_t，以前在NSURLConnection中相应的参数的类型是long long。
- 由于增加了completionHandler: 这个block参数，NSURLSession实际上给Foundation框架引入了一种全新的模式。这种模式允许delegate方法可以安全地在主线程运行，而不会阻塞主线程；Delegate只需要简单调用dispatch_async就可以切换到后台进行相关的的操作，然后在操作完成时调用completionHander即可。同时，它还可以有效地拥有多个返回值，而不需要我们使用笨拙的参数指针。以```NSURLSessionTaskDelegate```的```URLSession:task:didReceiveChallenge:completionHandler:```方法举例，```completionHandler```接受两个参数: ```NSURLSessionAuthChallengeDisposition```和```NSURLCredential```，前者为应对鉴权查询的策略，后者为需要使用的证书（仅当前者 -- 应对鉴权查询的策略为使用证书，即```NSURLSessionAuthChallengeUseCredential```时有效。否则该参数为 NULL）。

***2.3 NSURLSessionConfiguration*** <br>
NSURLSessionConfiguration 有三个类工厂方法：
>
- 1.```+defaultSessionConfiguration```返回一个标准的Configuration，这个配置实际上与NSURLConnection的网络堆栈（networking stack）是一样的，具有相同的共享NSHTTPCookieStorage，共享NSURLCache和共享NSURLCredentialStorage。
- 2.```+ephemeralSessionConfiguration```返回一个预设配置，这个配置中不会对缓存，Cookie和证书进行持久性的存储。这对于实现像密码浏览这种功能是很理想的。
- 3.```backgroundSessionConfiguration:(NSString *)identifier```的独特之处在于，它会创建一个后台session。后台session不同于常规的，普通的session，它甚至可以在应用程序挂起，退出或者奔溃的情况下运行上传和下载任务。初始化时指定的标识符，被用于向任何可能在进程外恢复后台传输的守护进程（daemon）提供上下文。

2.3.1 NSURLSessionConfiguration配置属性<br>
NSURLSessionConfiguration 拥有20个配置属性，熟练掌握这些配置属性的用处，可以让应用程序充分地利用网络环境。<br>
基本配置:
>
- ```HTTPAdditionalHeaders``` ：指定一组默认的可以设置出站请求（outbound request）的数据头。这对于跨session共享信息，如内容类型，语言，用户代理和身份认证，是很有作用的。
- ```networkServiceType```：对标准的网络流量，网络电话，语音，视频，以及由一个后台进程使用的流量进行区分。大多数应用程序都不需要设置这个。
- ```allowsCellularAccess```和```discretionary```：被用于节省通过蜂窝网络连接的带宽。对于后台传输的情况，推荐大家使用```discretionary```这个属性，而不是```allowsCellularAccess```，因为前者会把WiFi和电源的可用性考虑在内。
- ```timeoutIntervalForRequest```和```timeoutIntervalForResource```： 分别指定了对于请求和资源的超时间隔。许多开发人员试图使用```timeoutInterval```去限制发送请求的总时间，但其实它真正的含义是：分组（packet）之间的时间。实际上我们应该使用```timeoutIntervalForResource```来规定整体超时的总时间，但应该只将其用于后台传输，而不是用户实际上可能想要去等待的任何东西。
- ```HTTPMaximumConnectionsPerHost```： 是Foundation框架中URL在家系统的一个新的配置选项，它曾经被NSURLConnection用于管理私有的连接池。现在有了NSURLSession，开发者可以在需要时限制连接到特定主机的数量。
- ```HTTPShouldUsePipelining```：这个属性在```NSMUtableURLRequest```下也有，它可以被用于启用HTTP管线化（HTTP pipelining），这可以显著降低请求的加载时间，但是由于被服务器广泛支持，默认是禁用的。
- ```sessionSendsLaunchEvents```：是另一个新的属性，该属性指定session是否应该从后台启动。
- ```connectionProxyDictionary```：指定了session连接中的代理服务器，同样的，大多数面向消费者的应用程序都不需要代理，所以基本上不需要配置这个属性。

2.3.2 Cookie 策略<br>
- ```HTTPCookieStorage```： 存储了session所使用的cookie，默认情况下会使用```NSHTTPCookieShorage```的```sharedHTTPCookieStorage```这个单例对象，这与```NSURLConnection``` 是相同的。

- ```HTTPCookieAcceptPolicy``` ： 决定了什么情况下session应该接受从服务器发出的cookie。

- ```HTTPShouldSetCookies``` ：指定了请求是否应该使用session存储的cookie ，即```HTTPCookieSorage``` 属性的值。

2.3.3 安全策略<br>
- ```URLCredentialStorage```：存储了session所使用的证书，默认情况下会使用```NSURLCredentialStorage``` 的```+sharedCredentialStorage``` 这个单例对象，这与```NSURLConnection``` 是相同的。

- ```TLSMaximumSupportedProtocol``` 和```TLSMinimumSupportedProtocol``` 确定session是否主持SSL协议。

2.3.4 缓存策略 <br>
- ```URLCache``` 是session使用的缓存，默认情况下使用```URLCache```的```+sharedURLCache ```这个单例对象，这与```NSURLConnection``` 是相同的。

- ```requestCachePolicy``` 指定一个请求的缓存响应应该在什么时候返回。这相当于```NSURLRequest```的```-cachePolicy```方法。

2.3.5 自定义协议 <br>
- ```protocolClasses```用来配置特定某个session所使用的自定义协议（该协议是```NSURLProtocol```的子类）的数组。

### 二、计算机网络基础知识
计算机网络学习的核心就是网络协议的学习。网络协议是为计算机网络中进行数据交换而建立的规则，标准或者说是约定的集合。

**1、网络层次划分** <br>
国际标准化组织（ISO）在1978年提出了OSI/RM模型，它将计算机网络体系结构划分成七层，自下而上依次为：物理层（Physics Layer）、数据链路层（Data Link Layer）、网络层（Network Layer）、传输层（Transport Layer）、会话层（Session Layer）、表示层（Presentation Layer）、应用层（Application Layer）。其中四层完成数据传送服务，上面三层面向用户。<br>
除了标准的OSI七层模型以外，常见的网络层次划分还有TCP/IP四层协议以及TCP/IP五层协议，它们之前的关系如下如所示：

![网络层次关系图](/image/networkOSI.png "网络层次关系图")

**2、OSI七层网络模型和相关网络协议** <br>
互联网的任何操作都离不开网络协议，下图就表示了OSI七层模型中每一层有哪些协议：

![网络协议关系图](/image/networkProtocol.png "网络协议关系图")

**3、TCP/IP协议** <br>
&emsp;&emsp;TCP/IP协议是Internet最基本的协议、Internet国际互联网络的基础，由网络层的Ip协议和传输层的TCP协议组成。通俗而言：TCP负责发现传输的问题，一有问题就发出信号，要求重新传输，直到所有的数据安全正确地传输到目的地。而IP则是给Internet的每一台联网设备规定的一个地址。

1.IP层接收由更底层（网络接口层例如以太网设备驱动程序）发来的数据包，并把该数据包发送到更高层--TCP或UDP层；相反，IP层也把从TCP或UDP层接收来的数据包传送到更低层。IP数据包是不可靠的，因为IP并没有做任何事情来确认数据包是否按顺序发送的 或有没有被破坏，IP数据包中含有发送它的主机地址（源地址）和接收它的主机的地址（目的地址）。

2.TCP是面向连接的通信协议，通过三次握手建立连接，通讯完成时（通过四次握手）断开连接，由于TCP是面向连接的所以只能用于端到端的通讯。TCP提供的是一种可靠的数据流服务，采用“带重传的肯定确认”技术来实现传输的可靠性。TCP还采用了一种称为“滑动窗口”的方式进行流量控制，所谓的窗口实际表示接收能力，用以限制发送方的发送速度。

使用TCP协议：FTP（文件传输协议）、Telnet（远程登录协议）、SMTP（简单邮件传输协议）、POP3（和SMTP相对 ，用于接收邮件）、HTTP协议等。

**4、UDP协议** <br>
&emsp;&emsp;UDP用户数据报协议，是面向无连接的通讯协议，UDP数据包括端口号和源端口号信息，由于通讯不需要连接，所以可以实现广播发送。UDP通讯时不需要接收方确认，属于不可靠的传输，可能会出现丢包现象，实际应用中需要程序员验证。

- UDP与TCP位于同一层，但它不管数据包的顺序。错误或者重发，因此，UDP不被应用于那些使用虚电路的面向连接的服务，UDP主要用于那些面向查询--应答的服务，例如NFS。相对于FTP或Telnet，这些服务需要交换的消息量较小。

- 每个UDP报文分UDP报头和UDP数据区两部分。报头由四个16位（2字节）字段组成，分别说明该报文的源端口、目的端口，报文长度以及校验值。UDP报头由4个域组成，其中每个域各占两个字节。

使用UDP协议包括：TFTP（简单文件传输协议） 、SNMP（简单网络管理协议）、DNS（域名解析协议）、NFS、BOOTP

TCP 与UDP的区别： TCP是面向连接的，可靠的字节流服务；UDP是面向无连接的，不可靠的数据报服务。

**5、DNS协议** <br>
DNS 是域名系统（DomainNameSystem）缩写，该系统用于命名组织到域层次结构中的计算机和网络服务，可以简单的理解为将URL转换为IP地址。域名是圆点分开一串单词或缩写组成的，每一个域名都对应一个唯一的IP地址，在Interner上域名与IP地址之间是一一对应的，DNS就是进行域名解析的服务器。DNS命名用于Internet等TCP/IP网络中，通过用户友好的名称查找计算机和服务。

**6、NAT协议** <br>
NAT网络地址转换（Network Address Translation）属接入广域网（WAN）技术，是一种私有（保留）地址转化为合法IP地址的转换技术，它被广泛应用于各种类型Internet接入方式和各种类型的网络中。原因很简单，NAT不仅完美地解决了IP地址不足的问题，而且还能有效地避免来自网络外部的攻击，隐藏并保护网络内部的计算机。

**7、DHCP协议** <br>
DHCP动态主机设置协议（Dynamic Host Configuration）是一个局域网的网络协议，使用UDP协议工作，主要两个用途：给内部网络或网络服务供应商自动分配IP地址，给用户或者内部网络管理员作为对所有计算机作中央管理手段。

**8、HTTP协议** <br>
HTTP 超文本传输协议（HyperText Transfer Protocol）是互联网应用最为广泛的一种网络协议。所有的WWW文件都必须遵守这个标准。

HTTP协议包括哪些请求？
>
- GET : 请求读取由URL所标志的信息。
- POST：给服务器添加信息（如注释）。
- PUT：给定的URL下存储一个文档。
- DELETE：删除给定的URL所标志的资源。

HTTP中，POST与GET的区别？
>
- 1)、GET是从服务器上获取数据，POST是向服务器传送数据。
- 2)、GET是把参数数据队列加到提交表单的Action属性所指向的URL中，值和表单内各个字段一一对应，在URL中可以看到。
- 3)、GET传送量小，不能大于2KB；POST传送的数据量较大，一般默认不受限制。
- 4)、根据HTTP规范，GET用于信息获取，而且应该是安全的和幂等的。<br>
4.1 所谓的“安全的” 意味着,操作在于获取信息而非修改信息。换句话说，GET请求一般不产生副作用。就是说，它仅仅是获取资源信息，就像数据库查询一样，不会修改，增加数据，不会影响资源的状态。<br>
4.2 幂等 意味着对用一个URL的多个请求应该返回同样的结果。<br>
- 5)、POST请求比GET请求更加的安全，因为POST请求不会把信息添加在URL上的查询字符串上，但GET完全暴露，对于敏感信息必须使用POST。


**9、HTTPS - 安全的HTTP** <br>
Transport Layer Security (安全传输层协议，TLS) 是一种基于 TCP 的加密协议。它支持两件事：传输的两端可以互相验证对方的身份，以及加密所传输的数据。基于 TLS 的 HTTP 请求就是 HTTPS。

用 HTTPS 去替代 HTTP，在安全方面会有显著的提升。也许你还会采用一些其他的安全措施，总之这都会为安全通信提供保障。

### 三、总结

利用 NSURLSession 发 HTTP 请求是非常简单便捷的。但是请求背后有很多技术点做支撑。只有知晓和理解其中的细节和内涵才能更好的去优化 HTTP 请求。用户期望的是我们的 app 时时刻刻都是好用的。只有深刻理解 IP，TCP 和 HTTP 的工作原理才能更好的去满足用户的期望。上面的网络知识只是一些常识的基础，需要更深入的理解可以去查看[IP，TCP 和 HTTP讲解](https://objccn.io/issue-10-6/)这篇文章。

---
参考资料：<br>
[计算机网络基础](http://www.cnblogs.com/maybe2030/p/4781555.html) <br>
[@iOSGit资料](https://hit-alibaba.github.io/interview/iOS/Cocoa-Touch/Multithreading.html)<br>
[ObjC中国-NSURLSession](https://objccn.io/issue-5-4/)
