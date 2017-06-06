<p align="center">

<img src="Images/logo.gif" alt="Monitor" title="Monitor"/>

</p>

国内移动网络环境非常复杂，WIFI、4G、3G、2.5G(Edge)、2G 等多种移动网络并存，用户的网络可能会在 WIFI/4G/3G/2.5G/2G 类型之间切换，这是移动网络和传统网络一个很大的区别，被称作是 **Connection Migration** 问题。此外，还存在国内运营商网络的 DNS 解析慢、失败率高、DNS 被劫持的问题；还有国内运营商互联和海外访问国内带宽低传输慢等问题。这些网络问题令人非常头疼。移动网络的现状造成了用户在使用过程中经常会遇到各种网络问题，网络问题将直接导致用户无法在 App 进行操作，当一些关键的业务接口出现错误时，甚至会直接导致用户的大量流失。网络问题不仅给移动开发带来了巨大的挑战，同时也给网络监控带来了全新的机遇。以往要解决这些问题，只能靠经验和猜想，而如果能站在 App 的视角对网络进行监控，就能更有针对性地了解产生问题的根源。

网络监控一般通过 `NSURLProtocol` 和代码注入（Hook）这两种方式来实现，由于 `NSURLProtocol` 作为上层接口，使用起来更为方便，因此很自然选择它作为网络监控的方案，但是 `NSURLProtocol` 属于 **URL Loading System** 体系中，应用层的协议支持有限，只支持 **FTP**，**HTTP**，**HTTPS** 等几个应用层协议，对于使用其他协议的流量则束手无策，所以存在一定的局限性。监控底层网络库 `CFNetwork` 则没有这个限制。

下面是网络采集的关键性能指标：

* TCP 建立连接时间
* DNS 时间
* SSL 时间
* 首包时间
* 响应时间
* HTTP 错误率
* 网络错误率

### NSURLProtocol

``` objective-c
//为了避免 canInitWithRequest 和 canonicalRequestForRequest 出现死循环
static NSString * const HJHTTPHandledIdentifier = @"hujiang_http_handled";

@interface HJURLProtocol () <NSURLSessionTaskDelegate, NSURLSessionDataDelegate>

@property (nonatomic, strong) NSURLSessionDataTask *dataTask;
@property (nonatomic, strong) NSOperationQueue     *sessionDelegateQueue;
@property (nonatomic, strong) NSURLResponse        *response;
@property (nonatomic, strong) NSMutableData        *data;
@property (nonatomic, strong) NSDate               *startDate;
@property (nonatomic, strong) HJHTTPModel          *httpModel;

@end

+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    if (![request.URL.scheme isEqualToString:@"http"] &&
        ![request.URL.scheme isEqualToString:@"https"]) {
        return NO;
    }
    
    if ([NSURLProtocol propertyForKey:HJHTTPHandledIdentifier inRequest:request] ) {
        return NO;
    }
    return YES;
}

+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    
    NSMutableURLRequest *mutableReqeust = [request mutableCopy];
    [NSURLProtocol setProperty:@YES
                        forKey:HJHTTPHandledIdentifier
                     inRequest:mutableReqeust];
    return [mutableReqeust copy];
}

- (void)startLoading {
    self.startDate                                        = [NSDate date];
    self.data                                             = [NSMutableData data];
    NSURLSessionConfiguration *configuration              = [NSURLSessionConfiguration defaultSessionConfiguration];
    self.sessionDelegateQueue                             = [[NSOperationQueue alloc] init];
    self.sessionDelegateQueue.maxConcurrentOperationCount = 1;
    self.sessionDelegateQueue.name                        = @"com.hujiang.wedjat.session.queue";
    NSURLSession *session                                 = [NSURLSession sessionWithConfiguration:configuration delegate:self delegateQueue:self.sessionDelegateQueue];
    self.dataTask                                         = [session dataTaskWithRequest:self.request];
    [self.dataTask resume];

    httpModel                                             = [[NEHTTPModel alloc] init];
    httpModel.request                                     = self.request;
    httpModel.startDateString                             = [self stringWithDate:[NSDate date]];

    NSTimeInterval myID                                   = [[NSDate date] timeIntervalSince1970];
    double randomNum                                      = ((double)(arc4random() % 100))/10000;
    httpModel.myID                                        = myID+randomNum;
}

- (void)stopLoading {
    [self.dataTask cancel];
    self.dataTask           = nil;
    httpModel.response      = (NSHTTPURLResponse *)self.response;
    httpModel.endDateString = [self stringWithDate:[NSDate date]];
    NSString *mimeType      = self.response.MIMEType;
    
    // 解析 response，流量统计等
}

#pragma mark - NSURLSessionTaskDelegate

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    if (!error) {
        [self.client URLProtocolDidFinishLoading:self];
    } else if ([error.domain isEqualToString:NSURLErrorDomain] && error.code == NSURLErrorCancelled) {
    } else {
        [self.client URLProtocol:self didFailWithError:error];
    }
    self.dataTask = nil;
}

#pragma mark - NSURLSessionDataDelegate

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data {
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveResponse:(NSURLResponse *)response completionHandler:(void (^)(NSURLSessionResponseDisposition))completionHandler {
    [[self client] URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageAllowed];
    completionHandler(NSURLSessionResponseAllow);
    self.response = response;
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURLRequest * _Nullable))completionHandler {
    if (response != nil){
        self.response = response;
        [[self client] URLProtocol:self wasRedirectedToRequest:request redirectResponse:response];
    }
}

```

> **Hertz** 使用的是 `NSURLProtocol` 这种方式，通过继承 `NSURLProtocol`，实现 `NSURLConnectionDelegate` 来实现截取行为。

### Hook

如果我们使用手工埋点的方式来监控网络，会侵入到业务代码，维护成本会非常高。通过 Hook 将网络性能监控的代码自动注入就可以避免上面的问题，做到真实用户体验监控（RUM: Real User Monitoring），监控应用在真实网络环境中的性能。

> **AOP**(Aspect Oriented Programming，面向切面编程)，是通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态添加功能的一种技术。其核心思想是将业务逻辑（核心关注点，系统的主要功能）与公共功能（横切关注点，如日志、事物等）进行分离，降低复杂性，提高软件系统模块化、可维护性和可重用性。其中核心关注点采用 **OOP** 方式进行代码的编写，横切关注点采用 **AOP** 方式进行编码，最后将这两种代码进行组合形成系统。**AOP** 被广泛应用在日志记录，性能统计，安全控制，事务处理，异常处理等领域。

在 iOS 中 **AOP** 的实现是基于 **Objective-C** 的 **Runtime** 机制，实现 Hook 的三种方式分别为：**Method Swizzling**、**NSProxy** 和 **Fishhook**。前两者适用于 **Objective-C** 实现的库，如 `NSURLConnection` 和 `NSURLSession` ，**Fishhook** 则适用于 **C** 语言实现的库，如 `CFNetwork`。

下图是阿里百川码力监控给出的三类网络接口需要 hook 的方法

<p align="center">

<img src="Images/network_monitor.jpeg">

</p>


接下来分别来讨论这三种实现方式：

#### Method Swizzling

**Method swizzling** 是利用 **Objective-C** **Runtime** 特性把一个方法的实现与另一个方法的实现进行替换的技术。每个 Class 结构体中都有一个 `Dispatch Table` 的成员变量，`Dispatch Table` 中建立了每个 `SEL`（方法名）和对应的 `IMP`（方法实现，指向 **C** 函数的指针）的映射关系，**Method Swizzling** 就是将原有的 `SEL` 和 `IMP`映射关系打破，并建立新的关联来达到方法替换的目的。

<p align="center">

<img src="Images/method_swizzling.png">

</p>

因此利用 **Method swizzling** 可以替换原始实现，在替换的实现中加入网络性能埋点行为，然后调用原始实现。

#### NSProxy

> NSProxy is an abstract superclass defining an API for objects that act as stand-ins for other objects or for objects that don’t exist yet. Typically, a message to a proxy is forwarded to the real object or causes the proxy to load (or transform itself into) the real object. Subclasses of NSProxy can be used to implement transparent distributed messaging (for example, NSDistantObject) or for lazy instantiation of objects that are expensive to create.

这是 Apple 官方文档给 `NSProxy` 的定义，`NSProxy` 和 `NSObject` 一样都是根类，它是一个抽象类，你可以通过继承它，并重写 `-forwardInvocation:` 和 `-methodSignatureForSelector:` 方法以实现消息转发到另一个实例。综上，`NSProxy` 的目的就是负责将消息转发到真正的 target 的代理类。

**Method swizzling** 替换方法需要指定类名，但是 `NSURLConnectionDelegate` 和 `NSURLSessionDelegate` 是由业务方指定，通常来说是不确定，所以这种场景不适合使用 **Method swizzling**。使用 `NSProxy` 可以解决上面的问题，具体实现：proxy delegate 替换 `NSURLConnection` 和 `NSURLSession` 原来的 delegate，当 proxy delegate 收到回调时，如果是要 hook 的方法，则调用 proxy 的实现，proxy 的实现最后会调用原来的 delegate；如果不是要 hook 的方法，则通过消息转发机制将消息转发给原来的 delegate。下图示意了整个操作流程。

<p align="center">

<img src="Images/proxy.jpeg">

</p>

#### Fishhook

fishhook 是一个由 Facebook 开源的第三方框架，其主要作用就是动态修改 **C** 语言的函数实现，我们可以使用 fishhook 来替换动态链接库中的 **C** 函数实现，具体来说就是去替换 `CFNetwork` 和 `CoreFoundation` 中的相关函数。后面会在讲监控 `CFNetwork` 详细说明，这里不再赘述。

讲解完 iOS 上 hook 的实现技术，接下来讨论在 `NSURLConnection`、`NSURLSession` 和 `CFNetwork` 中，如何将上面的三种技术应用到实践中。

### NSURLConnection

<p align="center">

<img src="Images/urlconnection.jpeg">

</p>


### NSURLSession

<p align="center">

<img src="Images/urlsession.jpeg">

</p>

### CFNetwork

#### 概述

以 **NeteaseAPM** 作为案例来讲解如何通过 `CFNetwork` 实现网络监控，它是通过使用代理模式来实现的，具体来说，是在 `CoreFoundation` Framework 的 `CFStream` 实现一个 Proxy Stream 从而达到拦截的目的，记录通过 `CFStream` 读取的网络数据长度，然后再转发给 Original Stream，流程图如下：

<p align="center">

<img src="Images/cfnetwork_monitor.jpg" width="500">

</p>

#### 详细描述

由于 `CFNetwork` 都是 **C** 函数实现，想要对 **C** 函数 进行 Hook 需要使用 **Dynamic Loader Hook** 库函数 - [fishhook](https://github.com/facebook/fishhook)，

> **Dynamic Loader**（dyld）通过更新 **Mach-O** 文件中保存的指针的方法来绑定符号。借用它可以在 **Runtime** 修改 **C** 函数调用的函数指针。**fishhook** 的实现原理：遍历 `__DATA segment` 里面 `__nl_symbol_ptr` 、`__la_symbol_ptr` 两个 section 里面的符号，通过 Indirect Symbol Table、Symbol Table 和 String Table 的配合，找到自己要替换的函数，达到 hook 的目的。

`CFNetwork` 使用 `CFReadStreamRef` 做数据传递，使用回调函数来接收服务器响应。当回调函数收到流中有数据的通知后，将数据保存到客户端的内存中。显然对流的读取不适合使用修改字符串表的方式，如果这样做的话也会 hook 系统也在使用的 `read` 函数，而系统的 `read` 函数不仅仅被网络请求的 stream 调用，还有所有的文件处理，而且 hook 频繁调用的函数也是不可取的。

使用上述方式的缺点就是无法做到选择性的监控和 **HTTP** 相关的 `CFReadStream`，而不涉及来自文件和内存的 `CFReadStream`，**NeteaseAPM** 的解决方案是在系统构造 HTTP Stream 时，将一个 `NSInputStream` 的子类 `ProxyStream` 桥接为 `CFReadStream` 返回给用户，来达到单独监控 **HTTP Stream** 的目的。

<p align="center">

<img src="Images/cfnetwork_monitor_1.jpg" width="500">

</p>

具体的实现思路就是：首先设计一个继承自 `NSObject` 并持有 `NSInputStream` 对象的 **Proxy** 类，持有的 `NSInputStream` 记为 OriginalStream。将所有发向 Proxy 的消息转发给 OriginalStream 处理，然后再重写 `NSInputStream` 的 `read:maxLength:` 方法，如此一来，我们就可以获取到 stream 的大小了。
`XXInputStreamProxy` 类的代码如下：

``` objective-c
- (instancetype)initWithStream:(id)stream {
    if (self = [super init]) {
        _stream = stream;
    }
    return self;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [_stream methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    [anInvocation invokeWithTarget:_stream];
}
                                                        
```

继承 `NSInputStream` 并重写 `read:maxLength:` 方法：

``` objective-c
- (NSInteger)read:(uint8_t *)buffer maxLength:(NSUInteger)len {
    NSInteger readSize = [_stream read:buffer maxLength:len];
    // 记录 readSize
    return readSize;
}                                                   
```

`XX_CFReadStreamCreateForHTTPRequest` 会被用来替换系统的 `CFReadStreamCreateForHTTPRequest` 方法

``` objective-c

static CFReadStreamRef (*original_CFReadStreamCreateForHTTPRequest)(CFAllocatorRef __nullable alloc,
                                                                    CFHTTPMessageRef request);
                         
/**
 XXInputStreamProxy 持有 original CFReadStreamRef，转发消息到 original CFReadStreamRef，
 在 read 方法中记录获取数据的大小
 */
static CFReadStreamRef XX_CFReadStreamCreateForHTTPRequest(CFAllocatorRef alloc,
                                                           CFHTTPMessageRef request) {
    // 使用系统方法的函数指针完成系统的实现
    CFReadStreamRef originalCFStream = original_CFReadStreamCreateForHTTPRequest(alloc, request);
    // 将 CFReadStreamRef 转换成 NSInputStream，并保存在 XXInputStreamProxy，最后返回的时候再转回 CFReadStreamRef
    NSInputStream *stream = (__bridge NSInputStream *)originalCFStream;
    XXInputStreamProxy *outStream = [[XXInputStreamProxy alloc] initWithClient:stream];
    CFRelease(originalCFStream);
    CFReadStreamRef result = (__bridge_retained CFReadStreamRef)outStream;
    return result;
}                                                             
                                                        
```
使用 **fishhook** 替换函数地址

``` objective-c
void save_original_symbols() {
    original_CFReadStreamCreateForHTTPRequest = dlsym(RTLD_DEFAULT, "CFReadStreamCreateForHTTPRequest");
}                                                      
```

``` objective-c
rebind_symbols((struct rebinding[1]){{"CFReadStreamCreateForHTTPRequest", XX_CFReadStreamCreateForHTTPRequest, (void *)& original_CFReadStreamCreateForHTTPRequest}}, 1);                                                    
```

根据 `CFNetwork` API 的调用方式，使用 **fishhook** 和 Proxy Stream 获取 **C** 函数的设计模型如下：

<p align="center">

<img src="Images/cfnetwork_monitor_2.png" width="500">

</p>

### NSURLSessionTaskMetrics/NSURLSessionTaskTransactionMetrics

Apple 在 iOS 10 的 `NSURLSessionTaskDelegate` 代理中新增了 `-URLSession: task:didFinishCollectingMetrics:` 方法，如果实现这个代理方法，就可以通过该回调的 `NSURLSessionTaskMetrics` 类型参数获取到采集的网络指标，实现对网络请求中 DNS 查询/TCP 建立连接/TLS 握手/请求响应等各环节时间的统计。

``` objective-c
/*
 * Sent when complete statistics information has been collected for the task.
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didFinishCollectingMetrics:(NSURLSessionTaskMetrics *)metrics API_AVAILABLE(macosx(10.12), ios(10.0), watchos(3.0), tvos(10.0));
```

#### NSURLSessionTaskMetrics

`NSURLSessionTaskMetrics` 对象封装了 session task 的指标，每个 `NSURLSessionTaskMetrics` 对象有 `taskInterval` 和 `redirectCount` 属性，还有在执行任务时产生的每个请求/响应事务中收集的指标。

* `transactionMetrics`:`transactionMetrics` 数组包含了在执行任务时产生的每个请求/响应事务中收集的指标。

	``` objective-c
	/*
	 * transactionMetrics array contains the metrics collected for every request/response transaction created during the task execution.
	 */
	@property (copy, readonly) NSArray<NSURLSessionTaskTransactionMetrics *> *transactionMetrics;
	```

* `taskInterval`:任务从创建到完成花费的总时间，任务的创建时间是任务被实例化时的时间；任务完成时间是任务的内部状态将要变为完成的时间。

	``` objective-c
	/*
	 * Interval from the task creation time to the task completion time.
	 * Task creation time is the time when the task was instantiated.
	 * Task completion time is the time when the task is about to change its internal state to completed.
	 */
	@property (copy, readonly) NSDateInterval *taskInterval;
	```

* `redirectCount`:记录了被重定向的次数。

	``` objective-c
	/*
	 * redirectCount is the number of redirects that were recorded.
	 */
	@property (assign, readonly) NSUInteger redirectCount;
	```

#### NSURLSessionTaskTransactionMetrics

`NSURLSessionTaskTransactionMetrics` 对象封装了任务执行时收集的性能指标，包括了 `request` 和 `response` 属性，对应 HTTP 的请求和响应，还包括了从 ` fetchStartDate` 开始，到 `responseEndDate` 结束之间的指标，当然还有 `networkProtocolName` 和 `resourceFetchType` 属性。

* `request`:表示了网络请求对象。

	``` objective-c
	/*
	 * Represents the transaction request.
	 */
	@property (copy, readonly) NSURLRequest *request;
	```

* `response`:表示了网络响应对象，如果网络出错或没有响应时，`response` 为 `nil`。

	``` objective-c
	/*
	 * Represents the transaction response. Can be nil if error occurred and no response was generated.
	 */
	@property (nullable, copy, readonly) NSURLResponse *response;
	```

* `networkProtocolName`:获取资源时使用的网络协议，由 ALPN 协商后标识的协议，比如 h2, http/1.1, spdy/3.1。

	``` objective-c
	@property (nullable, copy, readonly) NSString *networkProtocolName;
	```

* `isProxyConnection`:是否使用代理进行网络连接。

	``` objective-c
	/*
	 * This property is set to YES if a proxy connection was used to fetch the resource.
	 */
	@property (assign, readonly, getter=isProxyConnection) BOOL proxyConnection;
	```

* `isReusedConnection`:是否复用已有连接。

	``` objective-c
	/*
	 * This property is set to YES if a persistent connection was used to fetch the resource.
	 */
	@property (assign, readonly, getter=isReusedConnection) BOOL reusedConnection;
	```

* `resourceFetchType`:`NSURLSessionTaskMetricsResourceFetchType` 枚举类型，标识资源是通过网络加载，服务器推送还是本地缓存获取的。

	``` objective-c
	/*
	 * Indicates whether the resource was loaded, pushed or retrieved from the local cache.
	 */
	@property (assign, readonly) NSURLSessionTaskMetricsResourceFetchType resourceFetchType;
	```

对于下面所有 `NSDate` 类型指标，如果任务没有完成，所有相应的 `EndDate` 指标都将为 `nil`。例如，如果 DNS 解析超时、失败或者客户端在解析成功之前取消，`domainLookupStartDate` 会有对应的数据，然而 `domainLookupEndDate` 以及在它之后的所有指标都为 `nil`。

这幅图示意了一次 HTTP 请求在各环节分别做了哪些工作

<p align="center">

<img src="Images/uslsessionmetrics.png">

</p>

如果是复用已有的连接或者从本地缓存中获取资源，下面的指标都会被赋值为 `nil`：

* domainLookupStartDate
* domainLookupEndDate
* connectStartDate
* connectEndDate
* secureConnectionStartDate
* secureConnectionEndDate

* `fetchStartDate`:客户端开始请求的时间，无论资源是从服务器还是本地缓存中获取。

	``` objective-c
	@property (nullable, copy, readonly) NSDate *fetchStartDate;
	```

* `domainLookupStartDate`:DNS 解析开始时间，Domain -> IP 地址。

	``` objective-c
	/*
	 * domainLookupStartDate returns the time immediately before the user agent started the name lookup for the resource.
	 */
	@property (nullable, copy, readonly) NSDate *domainLookupStartDate;
	```

* `domainLookupEndDate`:DNS 解析完成时间，客户端已经获取到域名对应的 IP 地址。

	``` objective-c
	/*
	 * domainLookupEndDate returns the time after the name lookup was completed.
	 */
	@property (nullable, copy, readonly) NSDate *domainLookupEndDate;
	```

* `connectStartDate`:客户端与服务器开始建立 TCP 连接的时间。

	``` objective-c
	/*
	 * connectStartDate is the time immediately before the user agent started establishing the connection to the server.
	 *
	 * For example, this would correspond to the time immediately before the user agent started trying to establish the TCP connection.
	 */
	@property (nullable, copy, readonly) NSDate *connectStartDate;
	```
	
	* `secureConnectionStartDate `:HTTPS 的 TLS 握手开始时间。
	
		``` objective-c
		/*
		 * If an encrypted connection was used, secureConnectionStartDate is the time immediately before the user agent started the security handshake to secure the current connection.
		 *
		 * For example, this would correspond to the time immediately before the user agent started the TLS handshake.
		 *
		 * If an encrypted connection was not used, this attribute is set to nil.
		 */
		@property (nullable, copy, readonly) NSDate *secureConnectionStartDate;
		```
		
	* `secureConnectionEndDate`:HTTPS 的 TLS 握手结束时间。
	
		``` objective-c
		/*
		 * If an encrypted connection was used, secureConnectionEndDate is the time immediately after the security handshake completed.
		 *
		 * If an encrypted connection was not used, this attribute is set to nil.
		 */
		@property (nullable, copy, readonly) NSDate *secureConnectionEndDate;
		```

* `connectEndDate`:客户端与服务器建立 TCP 连接完成时间，包括 TLS 握手时间。

	``` objective-c
	/*
	 * connectEndDate is the time immediately after the user agent finished establishing the connection to the server, including completion of security-related and other handshakes.
	 */
	@property (nullable, copy, readonly) NSDate *connectEndDate;
	```
	
* `requestStartDate `:开始传输 HTTP 请求的 header 第一个字节的时间。

	``` objective-c
	/*
	 * requestStartDate is the time immediately before the user agent started requesting the source, regardless of whether the resource was retrieved from the server or local resources.
	 *
	 * For example, this would correspond to the time immediately before the user agent sent an HTTP GET request.
	 */
	@property (nullable, copy, readonly) NSDate *requestStartDate;
	```
	
* `requestEndDate `:HTTP 请求最后一个字节传输完成的时间。

	``` objective-c
	/*
	 * requestEndDate is the time immediately after the user agent finished requesting the source, regardless of whether the resource was retrieved from the server or local resources.
	 *
	 * For example, this would correspond to the time immediately after the user agent finished sending the last byte of the request.
	 */
	@property (nullable, copy, readonly) NSDate *requestEndDate;
	```
	
* `responseStartDate`:客户端从服务器接收到响应的第一个字节的时间。

	``` objective-c
	/*
	 * responseStartDate is the time immediately after the user agent received the first byte of the response from the server or from local resources.
	 *
	 * For example, this would correspond to the time immediately after the user agent received the first byte of an HTTP response.
	 */
	@property (nullable, copy, readonly) NSDate *responseStartDate;
	```

* `responseEndDate`:客户端从服务器接收到最后一个字节的时间。

	``` objective-c
	/*
	 * responseEndDate is the time immediately after the user agent received the last byte of the resource.
	 */
	@property (nullable, copy, readonly) NSDate *responseEndDate;
	```
	
## Wrap up

iOS 的网络监控有两种实现方式：`NSURLProtocol` 和代码注入（Hook），文中给出了通过 `NSURLProtocol` 实现监控的具体实现，然后分别介绍了在 iOS 中如何使用 **Method Swizzling** 、**NSProxy** 和 **Fishhook** 进行 AOP Hook，文章也给出了三种 AOP Hook 技术在 `NSURLConnection`、`NSURLSession` 和 `CFNetwork` 的案例。最后详细介绍在 iOS 10 中新引入的 `NSURLSessionTaskMetrics` 和 `NSURLSessionTaskTransactionMetrics` 类，它们可以被用于获取网络相关的元数据，比如 DNS 查询、TLS 握手、请求响应等环节的耗时，这些数据可以帮助开发人员更好地分析网络性能。

## 参考资料

* [移动端性能监控方案 Hertz](http://tech.meituan.com/hertz.html)
* [NetworkEye](https://github.com/coderyi/NetworkEye)
* [netfox](https://github.com/kasketis/netfox)
* [网易 NeteaseAPM iOS SDK 技术实现分享](http://www.infoq.com/cn/articles/netease-ios-sdk-neteaseapm-technology-share)
* [Mobile Application Monitor IOS组件设计技术分享](http://bbs.netease.im/read-tid-149)
* [性能可视化实践之路](http://www.doc88.com/p-3072311816896.html)

