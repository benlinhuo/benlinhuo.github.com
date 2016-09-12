---
layout:     post
title:      Core Animation 基础知识总结 
category: iOS
tags: [iOS]
description: iOS 动画绝大部分都是通过 Core Animation 框架来完成的。Core Animation 会将大部分的实际绘图任务交给图形硬件来处理，图形硬件会加速图形渲染的速度。所以使用 Core Animation 制作的动画都拥有更高的帧率，而且显示效果更加平滑，当然它不会加重 CPU 的负担而影响程序的运行速度。
---

## 简介
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;APP 的开发，我们时常要观察的一个性能问题，就是流量的消耗。作为用户，当然是流量消耗越少越好，这样在非 WI-FI 下节省的就是钱啊；然而作为优秀的开发者，自然是尽其所能优化流量的使用。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们可以利用 Xcode 自带的流量测试工具进行测试：当我们 APP 处于运行状态时，我们使用 APP ，Xcode 在如下位置就可以看到流量使用的总和：

![Xcode Network](/assets/images/2016-07-12-network-traffic1.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们还可以通过 Charles 等工具抓包，查看到具体 API 请求的流量多少。当然如果我们想要模拟真实场景的使用，就需要通过嵌入流量统计代码到项目中，跟随用户使用 APP 情况，这样获取到的流量统计数据才最真实。它相对应的代码： [github 地址](https://github.com/benlinhuo/HBLNetowrkTraffic/tree/master)。这篇博客是它的补充。

## 简单的类图

### 类图关系


![简单类图关系](/assets/images/2016-07-12-network-traffic2.png)

### 类图解析
1. HBLNetworkObserver 是核心类，它有两个 category，用于补充该类的功能实现。该类实现了网络请求方法的交换，以及流量代码的嵌入。
2. 真正做数据统计的类是 HBLNetworkRecord，它接受 HBLNetworkObserver 这个类的调用，将流量数据统计的代码嵌入。
3. 当新创建一条流量统计数据或者请求结束时，发送通知给 HBLNetworkDataUpload 这个类，让其存储这条数据且回传服务器。HBLNetworkTransaction 是流量统计的 model 类
4. HBLNetworkTrafficManager 类管理了 HBLNetworkObserver 和 HBLNetworkDataUpload，用于统一对外的接口。


## 核心原理

### 如何嵌入流量统计代码

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们网络请求的库，绝大部分使用的是  NSURLSession，可能还有一些比较久远的 APP 使用的是 NSURLConnection，这两种我们都考虑。针对这两类库，我们利用 OC 的 runtime 机制，在所有网络请求之前，扫描项目中用到的所有类关于网络请求的方法，然后再 swizzle 这些 method（即交换两个方法的具体实现），在 swizzledMethod 中，我们先嵌入流量统计的代码，然后再完成该 method 本应该完成的任务。我们针对 NRURLSession 和 NSURLConnection 类方法和 delegate 分开进行 swizzle，原因很简单。部分代码如下：


* 需要捕获且嵌入流量统计代码的方法

``` 
+ (void)injectIntoAllNetworkDelegateClasses
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        const SEL selectors[] = {
            @selector(connectionDidFinishLoading:),
            @selector(connection:willSendRequest:redirectResponse:),
            @selector(connection:didReceiveResponse:),
            @selector(connection:didReceiveData:),
            @selector(connection:didFailWithError:),
            @selector(URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:),
            @selector(URLSession:dataTask:didReceiveData:),
            @selector(URLSession:dataTask:didReceiveResponse:completionHandler:),
            @selector(URLSession:task:didCompleteWithError:),
            @selector(URLSession:dataTask:didBecomeDownloadTask:delegate:),
            @selector(URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite:),
            @selector(URLSession:downloadTask:didFinishDownloadingToURL:)
        };
        
        const int numSelectors = sizeof(selectors) / sizeof(SEL);
        
        Class *classes = NULL;
        // 获取项目所有类的数目
        int numClasses = objc_getClassList(NULL, 0);
        
        if (numClasses > 0) {
            // classes 表示项目中的所有类
            classes = (__unsafe_unretained Class*)malloc(sizeof(Class) * numClasses);
            numClasses = objc_getClassList(classes, numClasses);
            
            // 循环所有类中的所有方法，只要某个类有 selectors 中的任何一个方法，就进行 swizzle method
            for (NSInteger classIdx = 0; classIdx < numClasses; classIdx++) {
                Class class = classes[classIdx];
                
                if (class == [HBLNetworkObserver class]) {
                    continue;
                }
                
                unsigned int methodCount = 0;
                Method *methods = class_copyMethodList(class, &methodCount);
                BOOL matchingSelectorFound = NO; // 用于结束一个类的循环
                
                for (unsigned int methodIdx = 0; methodIdx < methodCount; methodIdx++) {
                    for (int selectorIdx = 0; selectorIdx < numSelectors; selectorIdx++) {
                        if (method_getName(methods[methodIdx]) == selectors[selectorIdx]) {
                            [self injectIntoDelegateClass:class];
                            matchingSelectorFound = YES;
                            break;
                        }
                    }
                    if (matchingSelectorFound) {
                        break;
                    }
                }
                
                free(methods);
                
            }
            
            free(classes);
        }
        
        [self injectIntoNSURLConnectionCancel];
        [self injectIntoNSURLSessionTaskResume];
        
        [self injectIntoNSURLConnectionAsynchronousClassMethod];
        [self injectIntoNSURLConnectionSynchronousClassMethod];
        
        [self injectIntoNSURLSessionAsyncDataAndDownloadTaskMethods];
        [self injectIntoNSURLSessionAsyncUploadTaskMethods];
        
    });
}
```

* 方法的交换

```
// 使用 swizzle 的 block 实现替换已有方法 originSelector 的实现
+ (void)replaceImpOfOriginSelector:(SEL)originSelector Class:(Class)class swizzledBlock:(id)block swizzledSelector:(SEL)swizzledSelector
{
    Method originMethod = class_getInstanceMethod(class, originSelector);
    if (!originMethod) {
        return;
    }
    
    IMP implementation = imp_implementationWithBlock(block);
    class_addMethod(class, swizzledSelector, implementation, method_getTypeEncoding(originMethod));
    
    Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    method_exchangeImplementations(originMethod, swizzledMethod);
}
```

### 统计到的数据回传服务器
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们是每一条 API 请求作为一条数据存储。数据是使用文件存储在本地。我们最开始会将该文件中的所有数据读取到内存，然后每收到一条数据，先判断是否在 WI-FI 且条数已经达标，是的话就直接回传给服务器，请求成功清除文件和内存中数据，否则直接存文件即可。这个原理其实对于我们常用的 action log 回传服务器的机制是一样的，当然也可以有更好的机制。目前机制它会有数据丢失率，当我们想要发送的条数越多，丢失率就越高。


## 深入知识点
* 该库中使用了多线程。
  
  1. 在数据存储本地时，因为操作文件，会涉及到 IO 操作，所以可能会比较慢，为了不影响我们正常的业务代码执行，我专门开了个串行队列去做所有跟文件操作有关的事情。使用串行队列，还有个原因就是可以保证任务的执行顺序。
  2. 我们嵌入的流量统计代码部分，也是新开了一个串行队列去操作的，为的就是不妨碍正常的网络请求执行。假设流量统计代码和网络请求代码在一个线程中的话，就需要先执行完流量统计再执行网络请求，对正常业务造成影响。虽然，我们新开了一个线程，对一般业务来说，不会即使加入了流量统计代码也不会对其造成影响。但是如果在资源比较紧缺的时候，我们还新开一个线程，则可能会因为资源的占有而造成些许影响，这种是小概率事件。


* OC 的 runtime 机制
	1. 我们一个核心技术的应用就是使用 OC 运行时，我们可以交换两个方法的具体实现。这一技术还可以应用到其他场景，如我们添加 log 时，某些底层共用业务代码直接加 log 代码，会造成代码的入侵，所以我们可以通过 swizzle method 模式在上层业务代码中，交换方法实现来达到目的；还有就是，如果我们使用第三方库，第三方库的升级更改了某个方法的名称，而之前我们使用该方法很多，为了不需要直接替换，我们可以对这个方法进行 swizzle，以满足我们的需求。


## 开发遇到的问题记录
* 我们需要判断当前网络环境是否是 WI-FI，所以就写了如下的代码获取。主线程会被 block 的原因是在主线程中，需要同步执行获取网络状态的那个 block，而这个 block 又指定了必须要在主线程中执行，这就形成了我依赖于你，你又等待我，就死锁了。改过后的代码见 git 代码仓库。

```
[HBLNetworkUtility getNetworkStatus]; // 主线程执行该代码，主线程会被 block

+ (NSString *)getNetworkStatus
{
    __block NSString *networkStatus = @"";
    
    dispatch_sync(dispatch_get_main_queue(), ^{
    	 ......获取网络环境的代码    
    });
    return networkStatus;
}

```

## 不足之处
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前这个流量统计的 SDK 就只针对了网络请求使用了 NSURLConnection 和 NSURLSession 库的做了统计。如下情况，暂时还统计不到：使用 UIWebView 或者 WKWebView ，这是加载 H5 页面。最多我们通过使用 NSURLConnection 的方式加载 H5 页面可以统计到，但是该页面中的异步请求，比如图片、API 异步请求等，目前还统计不到。高德地图封装的 API 请求，查看了下，它底层使用的还是上述两种库之一，因此是可以统计到的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因为流量统计的条数如果比较多的话，数据量还是比较大的，所以最好我们可以先在本地进行压缩（目前一般使用的是 gzip）再传输。这个库里面没有做数据压缩，也是不足之一。
