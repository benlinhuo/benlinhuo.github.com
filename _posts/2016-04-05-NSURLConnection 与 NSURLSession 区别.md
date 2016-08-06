---
layout:     post
title:      NSURLConnection 与 NSURLSession 区别
category: iOS
tags: [iOS]
description: 它们二者都是苹果推荐的网络库。当它们推出 NSURLSession 时，是希望能替换 NSURLConnection，why？请看它们二者的区别
---

## 简介

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这两个网络接口大家都很熟悉。最新的 AFNetworking 3.0 开始已经弃用了 NSURLConnection ，改用了 NSURLSession。而且从 iOS7 推出 NSURLSession 以后，苹果已经在竭力推进 NSURLSession 的使用。iOS9 之后的 NSURLConnection 部分方法已经不支持了（被过期了）。那既然这样，NSURLSession 到底比 NSURLConnection 好在什么地方呢？ 

## 二者的比较

### 1. 后者支持 HTTP2.0
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们知道 HTTP2.0，相比于 HTTP1.x ，web 性能上有大大提升。如果我们哪天改用 HTTP2.0 了，然后接口又支持，那无形中就提升了 HTTP 请求的性能。NSURLConnection 是不支持的，NSURLSession 支持 HTTP2.0。

### 2. 任务上传和下载
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NSURLSession 专门针对数据的上传和下载，提供了解决方案，分别为：NSURLSessionDataTask，NSURLSessionUploadTask 和 NSURLSessionDownloadTask，他们都是 NSURLSession 的子类。其中的 NSURLSessionUploadTask 也是 NSRULSessionDataTask 的子类。因为 uploadTask 只不过在 HTTP 请求的时候，使用 POST 把数据放到 HTTP body 中。所以用 uploadTask 来做的事情，通常直接使用 dataTask 也是可以实现的，只是稍微麻烦点。不过，能使用封装好的 API 可以省下很多事情。 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这几个 task 创建后都是挂起状态的，需要 resume 才可以执行。当服务器返回的数据比较少的时候，NSURLSession 与 NSURLConnection 执行普通任务的操作步骤没有区别。

### 3. 下载任务的方式
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NSURLConnection 也实现了对文件的下载，不过它是先把整个文件下载到内存中，然后再写入沙盒，如果文件比较大，就容易出现内存暴涨，消耗量过大的情况。而实用 NSURLSessionUploadTask 下载文件时，会默认下载到沙盒中的临时文件夹tmp中，所以不会像 NSURLConnection 那样造成内存暴涨的现象。因为这是个临时文件夹，所以我们需要把文件拷贝到其他地方。下面这段代码来源于 AFNetworking。

```
#pragma mark - NSURLSessionDownloadTaskDelegate
// 当下载任务完成，调用
- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    NSError *fileManagerError = nil;
    self.downloadFileURL = nil;

    // 对下载的新文件进行处理：把下载的文件移动到自己指定的位置
    if (self.downloadTaskDidFinishDownloading) {
        self.downloadFileURL = self.downloadTaskDidFinishDownloading(session, downloadTask, location);
        if (self.downloadFileURL) {
            [[NSFileManager defaultManager] moveItemAtURL:location toURL:self.downloadFileURL error:&fileManagerError];

            if (fileManagerError) {
                [[NSNotificationCenter defaultCenter] postNotificationName:AFURLSessionDownloadTaskDidFailToMoveFileNotification object:downloadTask userInfo:fileManagerError.userInfo];
            }
        }
    }
}
```

### 4. 请求方法的控制
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NSURLConnection 是在实例化对象 connection 以后，通过调用 [connection start]; 发送请求。它可以使用 cancel ，可以对已经发送的 API 停止发送，不过一个不好的点是，在停止以后就不可以再继续访问之前创建然后又停止的那个 API 了，需要创建新的请求才可以。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果我们想要使用 NSURLConnection 来实现断点续传的话，它的关键是 HTTP reqeust 的头部字段 `Range` 。我们通过每次请求一部分数据，同时可以指定从哪个字节到哪个字节数据的下载，多次请求便可以实现断点续传了。这个字段的含义如下： 

```
Range头域可以请求实体的一个或者多个子范围。例如，
　　表示头500个字节：bytes=0-499
　　表示第二个500字节：bytes=500-999
　　表示最后500个字节：bytes=-500
　　表示500字节以后的范围：bytes=500-
　　第一个和最后一个字节：bytes=0-0,-1
　　同时指定几个范围：bytes=500-600,601-999
　　但是服务器可以忽略此请求头，如果无条件GET包含Range请求头，响应会以状态码206（PartialContent）返回而不是以200（OK）。
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NSURLSession 有三个控制方法：取消（cancel）、暂停（suspend）、继续（resume）。不过它在请求被暂停后是可以通过 `resume` 恢复请求任务的。所以使用 NSURLSession 实现断点续传的功能很简单。核心代码如下：

```
// 属性定义
@property (nonatomic, strong) NSURLSessionDownloadTask *task;
@property (nonatomic, strong) NSData *resumeData;
@property (nonatomic, strong) NSURLSession *session;


- (NSURLSession *)session
{
    if (!_session) {
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration defaultSessionConfiguration];
        self.session = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:[NSOperationQueue mainQueue]];
    }
    return _session;
}


// 暂停／继续下载
- (void)toggleDownload
{
    if (self.task == nil) {
        if (self.resumeData) {// 继续
            [self resumeDownload];
        } else {
            [self startDownload];// 开始下载
        }
    } else {
        [self pauseDownload];
    }
}

- (void)resumeDownload
{
    // 传入上次暂停下载返回的数据，就可以恢复下载
    self.task = [self.session downloadTaskWithResumeData:self.resumeData];
    // 开始任务
    [self.task resume];
    // 清空
    self.resumeData = nil;
}

- (void)startDownload
{
    // 1.创建一个下载任务
    NSURL *url = [NSURL URLWithString:@"http://m.hbl.com/xxx"];
    self.task = [self.session downloadTaskWithURL:url];
    // 2.开始任务
    [self.task resume];
}

- (void)pauseDownload
{
    __weak typeof(self) vc = self;
    [self.task cancelByProducingResumeData:^(NSData *resumeData) {
        //  resumeData : 包含了继续下载的开始位置\下载的url
        vc.resumeData = resumeData;
        vc.task = nil;
    }];
}

```

### 5. 后台下载
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NSURLSession 支持程序的后台下载和上传, 苹果官方将其称之进程之外的上传和下载, 这些任务都交给后台守护线程完成, 而不是应用本身, 即使文件在下载和上传过程中崩溃了也可以继续运行(当然如果用户强制关闭程序的话, NSURLSession会断开连接)。在前台，为了用户体验，下载过程中进度条会一直刷新进度。当程序进入后台后, 事实上任务是交给iOS系统来调度的, 具体什么时候下载完成就不知道了。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 如果是一个BackgroundSession，在Task执行的时候，用户切到后台，Session会和ApplicationDelegate做交互。当程序切到后台后，在BackgroundSession中的Task还会继续下载。如下分几个场景分析 Session 和 Application 的关系：

* 当加入了多个Task，程序没有切换到后台

	这种情况Task会按照NSURLSessionConfiguration的设置正常下载，不会和ApplicationDelegate有交互。
* 当加入了多个Task，程序切到后台，所有Task都完成下载。

	1> 在切到后台之后，Session的Delegate不会再收到，Task相关的消息，直到所有Task全都完成后，系统会调用ApplicationDelegate的`application:handleEventsForBackgroundURLSession:completionHandler:`回调，
	
	2> 之后“汇报”下载工作，对于每一个后台下载的Task调用Session的Delegate中的`URLSession:downloadTask:didFinishDownloadingToURL:`（成功的话）和`URLSession:task:didCompleteWithError:`（成功或者失败都会调用）。这两个 API 都是前台用于文件下载 delegate 方法。
	
	3> 之后调用Session的Delegate回调`URLSessionDidFinishEventsForBackgroundURLSession:`。
	
	注意：在ApplicationDelegate被唤醒后，会有个参数ComplietionHandler，这个参数是个Block，这个参数要在后面Session的Delegate中didFinish的时候调用一下，如下：
	
```

@implementation HBLAppDelegate
 
- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)())completionHandler
{
    BLog();
    /*
     Store the completion handler. The completion handler is invoked by the view controller's checkForAllDownloadsHavingCompleted method (if all the download tasks have been completed).
     */
      self.backgroundSessionCompletionHandler = completionHandler; // 下述方法 URLSessionDidFinishEventsForBackgroundURLSession: 需要执行该 block
}
//……
@end
 
//Session的Delegate
@implementation HBLViewController
 
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session
{
    APLAppDelegate *appDelegate = (APLAppDelegate *)[[UIApplication sharedApplication] delegate];
    if (appDelegate.backgroundSessionCompletionHandler) {
        void (^completionHandler)() = appDelegate.backgroundSessionCompletionHandler;
        appDelegate.backgroundSessionCompletionHandler = nil;
        completionHandler();
    }
 
    NSLog(@"All tasks are finished");
}
@end
```

* 当加入了多个Task，程序切到后台，下载完成了几个Task，然后用户又切换到前台。（程序没有退出）
	
	切到后台之后，Session的Delegate仍然收不到消息。在下载完成几个Task之后再切换到前台，系统会先汇报已经下载完成的Task的情况，然后继续下载没有下载完成的Task，后面的过程同第一种情况。
	
* 当加入了多个Task，程序切到后台，几个Task已经完成，但还有Task还没有下载完的时候关掉强制退出程序，然后再进入程序的时候。（程序退出了）

	最后这个情况比较有意思，由于程序已经退出了，后面没有下完Session就不在了后面的Task肯定是失败了。但是已经下载成功的那些Task，新启动的程序也没有听“汇报”的机会了。经过实验发现，这个时候之前在NSURLSessionConfiguration设置的NSString类型的ID起作用了，当ID相同的时候，一旦生成Session对象并设置Delegate，马上可以收到上一次关闭程序之前没有汇报工作的Task的结束情况（成功或者失败）。但是当ID不相同，这些情况就收不到了，因此为了不让自己的消息被别的应用程序收到，或者收到别的应用程序的消息，起见ID还是和程序的Bundle名称绑定上比较好，至少保证唯一性。

### 6. 配置信息
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NSURLSession 的构造方法（sessionWithConfiguration:delegate:delegateQueue:）中有一个 NSURLSessionConfiguration 类的参数可以设置配置信息，它决定了 cookie、安全性和高速缓存的策略，最大主机连接数，资源管理，网络超时等配置。而 NSURLConnection 不能进行这个配置，它是所有的都依赖于一个全局的配置对象，这样缺乏灵活性。这方面，NSURLSession 做的更好。

NSURLSession 的如下三种配置：

* +(NSURLSessionConfiguration *)defaultSessionConfiguration，配置信息使用基于硬盘的持久话Cache，保存用户的证书到钥匙串,使用共享cookie存储；

* +(NSURLSessionConfiguration *)ephemeralSessionConfiguration ，配置信息和default大致相同。除了，不会把cache，证书，或者任何和Session相关的数据存储到硬盘，而是存储在内存中，生命周期和Session一致。比如浏览器无痕浏览等功能就可以基于这个来做；

* +(NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier，配置信息可以创建一个可以在后台甚至APP已经关闭的时候仍然在传输数据的session。注意，后台Session一定要在创建的时候赋予一个唯一的identifier，这样在APP下次运行的时候，能够根据identifier来进行相关的区分。如果用户关闭了APP,IOS 系统会关闭所有的background Session。而且，被用户强制关闭了以后，IOS系统不会主动唤醒APP，只有用户下次启动了APP，数据传输才会继续。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AFNetworking 中创建的是一个共享 session。创建共享 session 好处如下：正常 HTTP 请求是需要建立 socket 连接的，也就是 TCP 连接的三次握手，对于 HTTP 这个短连接，它在请求发送回来后连接就会被断开，等下次新 API 请求时会再次三次握手建立 TCP 连接，之后的请求都是如此。但是如果我们使用了共享 session，则当我们第二次 API 请求的时候，就省了 TCP 三次握手的过程了，HTTP1.1 中出现了 Connection: keep-alive这个选项。这个优化选项，可以使得客户端和服务器端复用一个TCP连接，从而减小每次的网络请求时间。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;所以，共享的Session将会 `复用` TCP的连接，而每次都新建Session的操作将导致每次的网络请求都开启一个TCP的三次握手。


#### 备注：Keep-Alive 的使用（如下是摘取的相关介绍）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Keep-Alive功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，Keep-Alive功能避免了建立或者重新建立连接。市场上 的大部分Web服务器，包括iPlanet、IIS和Apache，都支持HTTP Keep-Alive。对于提供静态内容的网站来说，这个功能通常很有用。但是，对于负担较重的网站来说，这里存在另外一个问题：虽然为客户保留打开的连 接有一定的好处，但它同样影响了性能，因为在处理暂停期间，本来可以释放的资源仍旧被占用。当Web服务器和应用服务器在同一台机器上运行时，Keep- Alive功能对资源利用的影响尤其突出。 此功能为HTTP 1.1预设的功能，HTTP 1.0加上Keep-Aliveheader也可以提供HTTP的持续作用功能。

Keep-Alive: timeout=5, max=100

timeout：过期时间5秒（对应httpd.conf里的参数是：KeepAliveTimeout），max是最多一百次请求，强制断掉连接
就是在timeout时间内又有新的连接过来，同时max会自动减1，直到为0，强制断掉。

HTTP/1.0

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在HTTP/1.0版本中，并没有官方的标准来规定Keep-Alive如何工作，因此实际上它是被附加到HTTP/1.0协议上，如果客户端浏览器支持Keep-Alive，那么就在HTTP请求头中添加一个字段 Connection: Keep-Alive，当服务器收到附带有Connection: Keep-Alive的请求时，它也会在响应头中添加一个同样的字段来使用Keep-Alive。这样一来，客户端和服务器之间的HTTP连接就会被保持，不会断开（超过Keep-Alive规定的时间，意外断电等情况除外），当客户端发送另外一个请求时，就使用这条已经建立的连接
HTTP/1.1

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在HTTP/1.1版本中，官方规定的Keep-Alive使用标准和在HTTP/1.0版本中有些不同，默认情况下所在HTTP1.1中所有连接都被保持，除非在请求头或响应头中指明要关闭：Connection: Close  ，这也就是为什么Connection: Keep-Alive字段再没有意义的原因。另外，还添加了一个新的字段Keep-Alive:，因为这个字段并没有详细描述用来做什么，可忽略它
Not reliable（不可靠）

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;HTTP是一个无状态协议，这意味着每个请求都是独立的，Keep-Alive没能改变这个结果。另外，Keep-Alive也不能保证客户端和服务器之间的连接一定是活跃的，在HTTP1.1版本中也如此。唯一能保证的就是当连接被关闭时你能得到一个通知，所以不应该让程序依赖于Keep-Alive的保持连接特性，否则会有意想不到的后果

Keep-Alive和POST

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在HTTP1.1细则中规定了在一个POST消息体后面不能有任何字符，还指出了对于某一个特定的浏览器可能并不遵循这个标准（比如在POST消息体的后面放置一个CRLF符）。而据我所知，大部分浏览器在POST消息体后都会自动跟一个CRLF符再发送，如何解决这个问题呢？根据上面的说明在POST请求头中禁止使用Keep-Alive，或者由服务器自动忽略这个CRLF，大部分服务器都会自动忽略，但是在未经测试之前是不可能知道一个服务器是否会这样做。 










