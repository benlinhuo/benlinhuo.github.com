---
layout:     post
title:      SDURLCache 源码解析
category: iOS
tags: [iOS]
description: 看到这个库，主要是在研究网络接口缓存 -- NSURLCache的时候看到的。仔细看了下，它的作者竟然是 SDWebImage 的作者，于是就沉下心来仔细阅读源码。
---

## 简介

SDURLCache 是在研究网络接口缓存 -- NSURLCache 的时候看到的。仔细看了下，它的作者竟然是 SDWebImage 的作者，于是就沉下心来仔细阅读源码。一是解析它的缓存策略到底是怎么做的；二是借鉴学习其中关于存储时，它的多线程是怎么使用的，以及多线程间数据一致性是怎么保持的。

[github地址](https://github.com/rs/SDURLCache)

点开github地址，我们会发现这是个很老的库，代码提交大部分都是在七八年前了，也就是2010年左右，那时候的 iOS 还比较“老”，所以仔细看它的README文档，会发现一个问题：它也是继承于NSURLCache的一个子类去做的，但它的描述：

```
On iPhone OS, Apple did remove on-disk cache support for unknown reason. Some will say it's to save flash-drive life, others will arg it's to save disk capacity. As it is explained in the NSURLCacheStoragePolicy, the NSURLCacheStorageAllowed constant is always treated as NSURLCacheStorageAllowedInMemoryOnly and there is no way to force it back, the code is certainly gone on this platform. For whatever reason Apple removed this feature, you may be interested by having on-disk HTTP request caching in your application. SDURLCache gives back this feature to this iPhone OS for you.

在iPhone操作系统中，苹果删除了磁盘缓存的支持，原因不明。有些人说是因为节省闪存驱动器的寿命，还有人说是为了节省磁盘容量。在 NSURLCacheStoragePolicy（缓存策略）的官方解释中，NSURLCacheStorageAllowed 始终被视为 NSURLCacheStorageAllowedInMemoryOnly ，也就是说 iOS 中只有内存缓存，没有磁盘缓存。这些内容的代码已经在当前平台生效了，无法改变。不管苹果基于什么原因删除了这项功能，你肯定会对自己应用中使用磁盘缓存感兴趣。使用SDURLCache，磁盘缓存功能便可以在你的iPhone手机系统中使用了。

```

上述的描述，让我们很奇怪的一点是：NSURLCacheStorageAllowed 目前是可以正常做磁盘缓存的，为啥它说不行。结合历史来看，这个库的创建时间大概2010年左右，那时候，iOS 系统在iOS4左右，但在 iOS4.x 系统中，NSURLCache确实只有内存缓存，只有到 iOS5.x 及以上两者才都有，不过仅支持HTTP，HTTPS也还是在 iOS6 中才支持。

所以说这个库是比较古老了，我们一般来说不会直接拿来使用（因为这个库中磁盘缓存的实现，NSURLCache已经主动实现了。），但它基于本地存储，还是有研究价值的。后面如果自己实现本地数据的存储，不论是文件还是数据库，它多线程以及数据一致性的管理都可以借鉴学习，比如埋点数据的存储、传感器数据收集后本地的缓存等等场景。

在看一个源码的时候，我发现一个帮助理解的方法是：当我们单独看每个方法都差不多懂或者有点懂的时候，便可以自己通过创建个新文件，然后把源码中的方法按照自己理解的顺序，重新梳理下。这样可以帮助我们整理理顺逻辑。如下按照：初始化->存储->获取->删除。

## 缓存策略的设计

它这边是把 NSURLCache 当作内存缓存，然后基于 NSURLCache 再做磁盘缓存，磁盘缓存的逻辑是：把每个requestKey对应的response作为一个文件来存储（所有文件同一个目录下，目录外部调用者可指定），同时会存储一个配置文件 diskCacheInfo，通过这个配置文件可以获取到对应的存储信息，比如存储的时间和存储的response大小，因为需要控制磁盘空间大小，不能超过指定磁盘大小。response文件对应的文件名是cacheKey，所以可以直接文件名索引，便可以获得对应response内容。

在一个定期维护线程中（maintenanceTimer），磁盘存储空间达到峰值，缓存就会自动清除。所有的磁盘写入操作都在一个独立的线程中执行。

所以这边缓存策略的设计，从如下几个部分来解释（从上到下的逻辑）：

### 准备

#### 用于各种存储的索引cacheKey

通过查资料发现NSURLCache默认缓存只支持GET请求，再通过如下cacheKey生成的规则，猜测它也是只支持GET请求。否则，对于POST请求，它的参数是放在消息体中。所以对于一个request来说，如果只是拿它的url.absoluteString，则无法区别同一个URL不同参数的情况。所以个人认为 SDURLCache 就只支持GET请求（你可以把需要缓存的内容都设置为GET请求，一般也只有查询的接口【变动性差】适用于缓存）。

```
// 通过url.absoluteString
+ (NSString *)cacheKeyForURL:(NSURL *)url {
    const char *str = [url.absoluteString UTF8String];
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    CC_MD5(str, strlen(str), r);
    static NSString *cacheFormatVersion = @"2";
    return [NSString stringWithFormat:@"%@_%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x",
            cacheFormatVersion, r[0], r[1], r[2], r[3], r[4], r[5], r[6], r[7], r[8], r[9], r[10], r[11], r[12], r[13], r[14], r[15]];
}
```

#### canonicalRequestForRequest

它主要是把urlstring中存在的字符`#`给删除，因为URL中的 `#` 对服务端来说根本没用，它只是客户端的一个锚点，用于指定跳转到相应的位置。所以 `https://www.baidu.com/path#124` 和 `https://www.baidu.com/path#125` 会认为这2个是同一个API请求。

```
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request {
    NSString *string = request.URL.absoluteString;
    NSRange hash = [string rangeOfString:@"#"];
    if (hash.location == NSNotFound)
        return request;
    
    NSMutableURLRequest *copy = [request mutableCopy];
    copy.URL = [NSURL URLWithString:[string substringToIndex:hash.location]];
    return copy;
}
```

### 初始化

1. 公开初始化 SDURLCache 类的方法

```
- (id)initWithMemoryCapacity:(NSUInteger)memoryCapacity diskCapacity:(NSUInteger)diskCapacity diskPath:(NSString *)path {
    // 此处调用，是调用 NSURLCache 类的初始化方法，初始化内存和磁盘空间大小以及指定存储路径。还可以指定磁盘存储的策略 minCacheInterval 、ignoreMemoryOnlyStoragePolicy
    if ((self = [super initWithMemoryCapacity:memoryCapacity diskCapacity:diskCapacity diskPath:path])) {
        self.minCacheInterval = kAFURLCacheInfoDefaultMinCacheInterval;
        self.shouldRespectCacheControlHeaders = YES;
        self.diskCachePath = path;
        self.ignoreMemoryOnlyStoragePolicy = NO;
	}
    
    return self;
}
```

2. 初始化硬盘存储的路径，创建对应文件夹：

```
- (void)createDiskCachePath {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        NSFileManager *fileManager = [[NSFileManager alloc] init];
        if (![fileManager fileExistsAtPath:_diskCachePath]) {
            [fileManager createDirectoryAtPath:_diskCachePath
                   withIntermediateDirectories:YES
                                    attributes:nil
                                         error:NULL];
        }
    });
}
```

3. diskCacheInfo 

这个可以理解成将某一个requestkey的response 作为一个文件，存储到本地的配置文件。它本身就是个map，具体内容如下：

```
@{
    kAFURLCacheInfoAccessesKey: @{
    	cacheKey: 对应生成response文件的当前时间
    },
    kAFURLCacheInfoSizesKey: @{
    	cacheKey: 对应存储的文件size
    },
    kAFURLCacheInfoURLsKey: @{
    	cacheKey: 对应存储文件的原始urlString
    } // debug，用于调试
}

这个配置文件本身存储的路径是调用上述初始化方法指定的目录下，文件名是：kAFURLCacheInfoFileName。

```

diskCacheUsage 这个变量表示目前磁盘缓存已经存储的大小，它初始化是通过上述diskCacheInfo变量下，这个key值kAFURLCacheInfoSizesKey所有数据之和。

它会创建一个maintenanceTimer，它的作用是在完成一个requestKey对应的response存储之后，去检查存储之后的磁盘空间是否超过了给定的磁盘存储空间。检查过一次之后，它会暂时挂起自己，等到下一次再往磁盘存储东西后会再次启动去检查（磁盘空间发生变化了）。当检查到超过了指定的存储空间，会按照一定的规则删除，这个会在删除中说明。

### 存储（重点）

1. diskCacheInfo 存文件 

```
// 此处使用 NSPropertyListSerialization 来存储，主要是让NSDictionary数据存储到文件后是二进制数据，不可读，提高安全性。
 NSData *data = [NSPropertyListSerialization dataFromPropertyList:self.diskCacheInfo format:NSPropertyListBinaryFormat_v1_0 errorDescription:NULL];
        if (data) {
            [data writeToFile:[_diskCachePath stringByAppendingPathComponent:kAFURLCacheInfoFileName] atomically:YES];
        }
```

2. 重写NSURLCache 的方法 `- (void)storeCachedResponse:(NSCachedURLResponse *)cachedResponse forRequest:(NSURLRequest *)request`

它在NSURLCache的该方法中增加了磁盘缓存。

首先它会判断该request是否应该缓存：

```
if (!_allowCachingResponsesToNonCachedRequests &&
        (request.cachePolicy == NSURLRequestReloadIgnoringLocalCacheData
         || request.cachePolicy == NSURLRequestReloadIgnoringLocalAndRemoteCacheData
         || request.cachePolicy == NSURLRequestReloadIgnoringCacheData)) {
            // When cache is ignored for read, it's a good idea not to store the result as well as this option
            // have big chance to be used every times in the future for the same request.
            // NOTE: This is a change regarding default URLCache behavior
            // 此次不该进行缓存
            return;
        }
```

当可以进行缓存的时候，先调用 `[super storeCachedResponse:cachedResponse forRequest:request];
` 进行内存缓存。

再根据一些选项，判断当前是否应该进行磁盘缓存，这其中涉及到对后台返回数据头信息的解析，包括cache-control, Last-Modified等，但其实仔细阅读代码，会发现它的判断和现在我们的认知会有不同，比如Expire我们基本不会用了，且有些情况我们会使用Etag，所以这个我们归属于历史问题。

上述判断后认为我们应该做磁盘缓存，则真正做磁盘缓存的方法是 `- (void)storeRequestToDisk:(NSURLRequest *)request response:(NSCachedURLResponse *)cachedResponse`。下面解析

3. 磁盘缓存

`- (void)storeRequestToDisk:(NSURLRequest *)request response:(NSCachedURLResponse *)cachedResponse `此方法分为3部分：1）存储request对应的response文件，且文件名是使用cacheKey命名，方便查找；2）diskCacheInfo 中的信息（文件生成时间+文件大小+对应的原始urlString）；3）存储之后需要检查是否超过了规定的磁盘空间（maintenanceTimer）。

### 获取

1. `- (NSCachedURLResponse *)cachedResponseForRequest:(NSURLRequest *)request`

该方法也是NSURLCache对外提供的一个方法，此处相当于重写了。

1）先查看内存缓存；2）无内存缓存的话，查找磁盘缓存：先查看diskCacheInfo 中是否有该request的信息，如有，则获取对应的文件内容，同时要重新生成时间，因为获取就相当于使用了，按照最近最少使用的规则，时间变成当前获取的时间，然后在下次定期维护线程中去存储这个时间（不立即存储，是因为文件内容没有发生任何变化，只是时间变了，只要在下次维护时即可。）

### 删除

1. 当检查当前已用磁盘空间 > 给定的磁盘存储空间

	当检查当前已用磁盘空间 > 给定的磁盘存储空间（代码对应的方法是 balanceDiskUsage），会根据LRU的原则，将最近最少使用的资源删除。

```
- (void)balanceDiskUsage {
    if (_diskCacheUsage < self.diskCapacity) {
        return; // Already done
    }
    
    dispatch_async_afreentrant(get_disk_cache_queue(), ^{
        NSMutableArray *keysToRemove = [NSMutableArray array];
        
        // 将最近最少使用的资源移出缓存空间
        // Apply LRU cache eviction algorithm while disk usage outreach capacity
        NSDictionary *sizes = [self.diskCacheInfo objectForKey:kAFURLCacheInfoSizesKey];
        
        // keysSortedByValueUsingSelector 按照排序来表示最近最少使用，因为存储的value值是存储response文件的时间。
        NSInteger capacityToSave = _diskCacheUsage - self.diskCapacity;
        NSArray *sortedKeys = [[self.diskCacheInfo objectForKey:kAFURLCacheInfoAccessesKey] keysSortedByValueUsingSelector:@selector(compare:)];
        NSEnumerator *enumerator = [sortedKeys objectEnumerator];
        NSString *cacheKey;
        
        // 此次倒序遍历，删除直到某次磁盘存储空间<指定存储空间
        while (capacityToSave > 0 && (cacheKey = [enumerator nextObject])) {
            [keysToRemove addObject:cacheKey];
            capacityToSave -= [(NSNumber *)[sizes objectForKey:cacheKey] unsignedIntegerValue];
        }
        
        [self removeCachedResponseForCachedKeys:keysToRemove];
        [self saveCacheInfo];
    });
}
```

2. 删除指定的 cacheKeys 对应的reponse object

`- (void)removeCachedResponseForCachedKeys:(NSArray *)cacheKeys`该方法它删除对应的response object。它首先需要遍历，在每一个cacheKey对应下，需要删除4个部分：diskCacheInfo 中存储的3部分数据（response文件存储时间+文件大小+对应的urlstring），还有就是response 内容的文件（文件删除通过遍历目录下的所有文件进行匹配）。

```
// 遍历cacheKey
            while((cacheKey = [enumerator nextObject])) {
                NSUInteger cacheItemSize = [[sizes objectForKey:cacheKey] unsignedIntegerValue];
                [accesses removeObjectForKey:cacheKey];
                [sizes removeObjectForKey:cacheKey];
#if SDURLCACHE_DEBUG
                [urls removeObjectForKey:cacheKey];
#endif
                [fileManager removeItemAtPath:[_diskCachePath stringByAppendingPathComponent:cacheKey] error:NULL];
                // 这么看diskCacheUsage是所有sizes数组之和
                _diskCacheUsage -= cacheItemSize;
            }
```

3. 删除指定的request对应的缓存

这部分删除，则需要删除对应的内存缓存和磁盘缓存。

```
- (void)removeCachedResponseForRequest:(NSURLRequest *)request {
    request = [[self class] canonicalRequestForRequest:request];
    // 删除对应的内存缓存
    [super removeCachedResponseForRequest:request];
    // 删除对应的磁盘缓存
    [self removeCachedResponseForCachedKeys:[NSArray arrayWithObject:[[self class] cacheKeyForURL:request.URL]]];
    [self saveCacheInfo];
}
```

需要指明的是：SDURLCache 中提供的方法，如上述介绍的存储、获取等一些方法，都是重写的SDURLCache中的方法，所以当我们用如下代码设置sharedURLCache后，在该缓存的地方，系统会自动调用SDURLCache中的对应方法。

如下是在网上找到的一段使用SDURLCache库的代码：

```
SDURLCache *urlCache = [[SDURLCache alloc]
                        initWithMemoryCapacity:1024*1024*2   // 2MB mem cache
                        diskCapacity:1024*1024*15 // 15MB disk cache
                        diskPath:[SDURLCache defaultCachePath]];
[urlCache setMinCacheInterval:1];
[NSURLCache setSharedURLCache:urlCache];

NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:@"http://www.baidu.com"]];
AFHTTPRequestOperation *operation = [[AFHTTPRequestOperation alloc] initWithRequest:request];
[operation setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
    NSLog(@"从服务端获取数据：%@",operation.responseString);
} failure:^(AFHTTPRequestOperation *operation, NSError *error) {
    if ([urlCache isCached:[NSURL URLWithString:@"http://www.baidu.com"]]) {
        
    }
    NSCachedURLResponse *resp = [urlCache cachedResponseForRequest:request];
    NSString  *str = [[NSString alloc] initWithData:resp.data encoding:NSUTF8StringEncoding];
    NSLog(@"从缓存中获取数据：%@",str);
}];
[operation start];  
```



## 源码中可借鉴学习的点

### 队列创建的方式

一般我自己创建队列的方式，是把队列变量作为类的一个属性。但该库中不是，它创建和引用方式（同苹果GCD提供的API）：

```
// 创建一个静态C方法
static dispatch_queue_t get_disk_cache_queue() {
    static dispatch_once_t onceToken;
    static dispatch_queue_t _diskCacheQueue;
    dispatch_once(&onceToken, ^{
        _diskCacheQueue = dispatch_queue_create("com.petersteinberger.disk-cache.processing", NULL); // NULL 表示串行队列
    });
    return _diskCacheQueue;
}

// 如何使用
dispatch_async(get_disk_cache_queue(), ^{
	... block中需要执行的内容
});
```

### 指定队列执行内容

指定队列执行内容和指定线程执行，一样的道理：

```
void dispatch_async_afreentrant(dispatch_queue_t queue, dispatch_block_t block);
inline void dispatch_async_afreentrant(dispatch_queue_t queue, dispatch_block_t block) {
	// 不过dispatch_get_current_queue()方法因为怕被误用，所以iOS 6之后便被废弃了。
	// https://blog.csdn.net/yiyaaixuexi/article/details/17752925 解释了被废弃的具体原因
    dispatch_get_current_queue() == queue ? block() : dispatch_async(queue, block);
}

// 指定线程执行，比如主线程
if ([[NSThread currentThread] isMainThread]) {
       block();
    } else {
        dispatch_async(dispatch_get_main_queue(), ^{
            block();
        });
    }
```

### 子线程在存储中的运用

代码中可以看到它创建了2个队列（都是串行队列）：get_disk_io_queue 和 get_disk_cache_queue。

在我们创建文件目录时(`createDiskCachePath`)，或者生成request对应response文件时(`[NSKeyedArchiver archiveRootObject:cachedResponse toFile:cacheFilePath]`)，它是在队列 get_disk_io_queue 中执行。

然后其他和缓存相关的操作都是在队列 get_disk_cache_queue 中。其中有：1）maintenanceTimer 的挂起；2）diskCacheInfo 对应文件内容的读取，以及读取后的已使用空间的求和等操作；3）diskCacheInfo  内容存储到文件中 `saveCacheInfo`；4）删除缓存，包括删除对应response 文件 `removeCachedResponseForCachedKeys` 等等。

其实整体对比下来，使用这2个不同队列的地方，只是从功能上来划分了，其实从个人角度来说，不太有必要使用2个串行队列，一个就好了。因为线程的切换也是需要资源的，而且对于同一个response文件的创建和删除还不在同一个串行队列。对于上述源码的阅读和理解，使用一个串行队列即可。

上述均是自己的解释，如有任何不妥，可探讨。



