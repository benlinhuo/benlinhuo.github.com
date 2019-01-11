---
layout:     post
title:      NSURLCache 网络接口缓存
category: iOS
tags: [iOS]
description: 一般的网络请求框架，没人不知道AFNetworking，但是它并没有封装现成可用的网络接口缓存。一个对用户体验高要求的APP来说，接口数据的缓存必不可少，一般这种网络请求的缓存，有些可能使用文件来缓存。但苹果提供了一个现成的方案 -- NSURLCache 。
---

## NSURLCache

一般的网络请求框架，没人不知道AFNetworking，但是它并没有封装现成可用的网络接口缓存。一个对用户体验高要求的APP来说，接口数据的缓存必不可少，一般这种网络请求的缓存，有些可能使用文件来缓存。但苹果提供了一个现成的方案 -- NSURLCache 。下面将提供一个方案来完成接口缓存的实现。注意[NSURLCache sharedURLCache]是系统提供的单例，使用很简单。


### 代码实现

此处提供一个完整代码的实现（曾经项目中使用过）。其实利用 `NSURLCache` 来做这个事情，还是很简单的。自己制定规则读写一个带有 NSURLResponse对象的NSURLCache，创建一个 `NSURLCache` 的子类。

其实这个类很简单，唯一做的2件事是：1) 它根据自己的需求生成存储的 requestKey，而不只是url.absoluteString；2）它根据自己的需求在适当的时机进行增删改。

```
#import "HBLURLCache.h"

static NSUInteger memoryCacheSize = 100 * 1024 * 1024;
static NSUInteger diskCacheSize = 100 * 1024 * 1024;

static NSUInteger HTTPReturnSuccess = 0;
static NSUInteger HTTPReturnSuccessCode = 200;


@implementation HBLURLCache

+ (instancetype)sharedCache {
    static dispatch_once_t onceToken;
    static HBLURLCache *cacheInstance = nil;
    dispatch_once(&onceToken, ^{
        cacheInstance = [[HBLURLCache alloc] initWithMemoryCapacity:memoryCacheSize diskCapacity:diskCacheSize diskPath:@"networkCacheDisk"];
        // 如无特殊要求，可以直接获取 [NSURLCache sharedURLCache]; 默认的URLCache
        // 若觉得默认满足不了要求，则可以通过 [NSURLCache setSharedURLCache:cacheInstance]; 设置
        // 注：setSharedURLCache 使用时需要确认没有其他调用者使用之前的单例cache，因为对于已存储的数据，如果再重新设置新的cache单例，则会对已存储的数据造成不可挽回的后果。
        // 所以即使没有这2句话，iOS 也会自动参与缓存，只不过是系统自动创建的NSURLCache类，可以通过 [NSURLCache sharedURLCache]; 获取
        [NSURLCache setSharedURLCache:cacheInstance];
        
    });
    return cacheInstance;
}

- (NSURLRequest *)generateStoredRequest:(NSURLRequest *)originalURL {
    NSMutableString *bodyString = [[NSMutableString alloc] initWithData:originalURL.HTTPBody encoding:NSUTF8StringEncoding];
    // 头信息中的sign是根据用户account+设备id+固定字符串，生成的md5串
    NSString *urlString = [NSString stringWithFormat:@"%@_%@_%@", [originalURL.URL absoluteString], [originalURL valueForHTTPHeaderField:@"sign"], bodyString];
    // 生成存储的request:同一台设备+同一个用户+同一个请求+同一个请求参数，算作是同一个请求
    NSMutableURLRequest *storedRequest = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:urlString] cachePolicy:NSURLRequestUseProtocolCachePolicy timeoutInterval:10.f];
    return storedRequest;
}

#pragma mark - remove
- (void)removeCacheForRequest:(NSURLRequest *)request {
    [[HBLURLCache sharedCache] removeCachedResponseForRequest:[self generateStoredRequest:request]];
}

#pragma mark - setter
- (void)storeCache:(id)responseObject
          response:(NSURLResponse *)response
           request:(NSURLRequest *)request {
    NSString *returnCode = nil;
    if ([responseObject isKindOfClass:[NSDictionary class]]) {
        returnCode = [responseObject objectForKey:@"returnCode"];
    }
    if (!returnCode || [returnCode respondsToSelector:@selector(integerValue)]) {
        return;
    }
    // 业务上失败的数据不缓存
    if(returnCode.integerValue != HTTPReturnSuccess && HTTPReturnSuccessCode != returnCode.integerValue) {
        return;
    }
    NSData *data = [NSKeyedArchiver archivedDataWithRootObject:responseObject];
    HBLURLCache *cache = [HBLURLCache sharedCache];
    NSCachedURLResponse *urlCacheResponse = [[NSCachedURLResponse alloc] initWithResponse:response data:data];
    
    [cache storeCachedResponse:urlCacheResponse forRequest:[self generateStoredRequest:request]];
}

- (void)storeCache:(id)responseObject dataTask:(NSURLSessionDataTask *)dataTask {
    [self storeCache:responseObject response:dataTask.response request:dataTask.originalRequest];
}

#pragma mark - getter
- (id)cacheForRequest:(NSURLRequest *)request {
    HBLURLCache *cache = [HBLURLCache sharedCache];
    // cachedResponseForRequest NSURLCache的方法获取缓存的数据
    NSCachedURLResponse *response = [cache cachedResponseForRequest:[self generateStoredRequest:request]];
    NSData *data = response.data;
    NSDictionary *cachedInfo = nil;
    @try {
        cachedInfo = [NSKeyedUnarchiver unarchiveObjectWithData:data];
    } @catch(NSException *exception) {}
    
    if (!cachedInfo) {
        [self removeCacheForRequest:request];
    }
    return cachedInfo;
}

- (id)cacheForDataTask:(NSURLSessionDataTask *)dataTask {
    return [self cacheForRequest:dataTask.originalRequest];
}

@end
```

上述缓存的原理：

1. 在第一次从服务端请求回来数据后，会根据请求的request生成一个存储的request（不使用默认的，是因为我们需要根据自己的维度来存储该请求，目前是同一台设备+同一个用户+同一个请求+同一个请求参数，算作是同一个请求。方法 generateStoredRequest：）
2. 当同一个请求，第二次请求的时候，我们会根据业务需求，在刚请求的时候，就可以直接拿缓存数据直接返回（isPlaceholder为YES的情况下），然后等数据回来再刷新一次（同时更新该接口缓存数据）；如果isPlaceholder为NO，则在数据回来的时候，如果接口返回失败的，则可以拿缓存数据展示（不过这种需求基本好像不会被使用）

上述其中方法 `setSharedURLCache:` 其实也是创建单例，只不过它通过传入一个自定义的本类对象来创建，创建之后仍然可以通过 `[NSURLCache sharedURLCache];` 来获取。所以我们创建单例对象也可以通过外部创建。

### NSURLCache 介绍

默认情况下，NSURLCache 是会有默认缓存的，且只对GET请求进行缓存，因为它认为GET一般都是查询类的数据，变动性不强。它的缓存原理是：一个NSURLRequest 对应一个 NSCachedURLResponse，我们上述的代码实现原理也是如此。


#### 缓存原理

它缓存的数据，是使用数据库存在本地。我们打开沙盒路径下的 `Library/Caches` ，如果像上述代码指定存储路径，则会在 `Library/Caches/networkCacheDisk` 下有如下数据库文件：

![数据库文件路径](/assets/images/urlcache.png)


我们可以在 Appstore 直接下载sqlite的打开工具 `Datum - Lite`，则会看到该数据库中有多张表。

1. 表cfurl_cache_response
`cfurl_cache_response` 表中可以看到有个字段 `request_key`，它的值便是我们存入时指定的request，针对上述代码，则为 `https://mgw-daily.xxx.com/appapi/za-dm-skywalker/v1/app/policy/list/claimable_376B8B6C5B2502750F4A341956F9ADFD_%7B%22searchType%22:%22all%22,%22familyMemberShip%22:%220%22,%22pageSize%22:%2210%22,%22pageNo%22:%221%22%7D`，拆开可以看到时请求URL+sign+请求参数拼接而成。

![cfurl_cache_response内容](/assets/images/cfurl_cache_response.png)


其中有个字段storage_policy:缓存策略。它主要用于标明是否缓存，其中0表示 NSURLCacheStorageAllowed

```
typedef NS_ENUM(NSUInteger, NSURLCacheStoragePolicy)
{
    NSURLCacheStorageAllowed,   //内存，磁盘都可以存
    NSURLCacheStorageAllowedInMemoryOnly,   //只能在内存中
    NSURLCacheStorageNotAllowed, //不允许
};

```

![cfurl_cache_response内容](/assets/images/cfurl_cache_response_field.png)


2. 表cfurl_cache_blob_data  

![cfurl_cache_blob_data内容](/assets/images/cfurl_cache_blob_data.png)

上图可以看到该表所有字段，entry_ID是主键，request_object即请求对象，response_object即响应对象，都是BLOB类型。

当我们通过上述打开sqlite的工具，可以右击"Save To File"，我们把保存后的文件后缀更改为plist，通过Xcode打开，发现它就是个plist文件，我们可以查看里面内容，见下图。会发现它里面包含了request和response的相关信息

![blob的内容](/assets/images/blob_plist.jpg)


3. 表 cfurl_cache_receiver_data

![cfurl_cache_receiver_data的内容](/assets/images/cfurl_cache_receiver_data.jpg)

上图发现，它只有3个字段，其中字段receiver_data，便是存储的一些服务端返回的数据，可能是json、图片、资源文件等等。


多张表中，比较关键的就是上述3张表，表中缓存查找的逻辑：

1. 在 `cfurl_cache_response ` 中根据requestKey查找到entry_ID;
2. 在 `cfurl_cache_blob_data `中根据entry_ID 找到response_object；
3. 在 `cfurl_cache_receiver_data `中根据entry_ID找到 receiver_data;

待我们拿到上述的response_object和receiver_data，便可以拼装 NSURLCachedURLResponse;

```
NSURLResponse *urlResponse = [[NSURLResponse alloc] initWithURL:request.URL MIMEType:[[request allHTTPHeaderFields] objectForKey:@"Accept"] expectedContentLength:[(NSData *)response_object length] textEncodingName:nil];

NSCachedURLResponse *cachedURLResponse = [[NSCachedURLResponse alloc] initWithResponse:urlResponse data:receiver_data userInfo:nil storagePolicy:NSURLCacheStorageAllowed];

```


#### 常见用法

NSURLCache 的常见用法可参见如下：

1. 获得全局缓存对象（没必要手动创建）NSURLCache *cache = [NSURLCache sharedURLCache]; 

2. 设置内存缓存的最大容量（字节为单位，默认为512KB）- (void)setMemoryCapacity:(NSUInteger)memoryCapacity;

3. 设置硬盘缓存的最大容量（字节为单位，默认为10M）- (void)setDiskCapacity:(NSUInteger)diskCapacity;

4. 硬盘缓存的位置：沙盒/Library/Caches

5. 取得某个请求的缓存- (NSCachedURLResponse *)cachedResponseForRequest:(NSURLRequest *)request; 

6. 清除某个请求的缓存- (void)removeCachedResponseForRequest:(NSURLRequest *)request;

7. 清除所有的缓存- (void)removeAllCachedResponses;

需要说明的是：NSURLCache 可以拦截UIWebView的任何请求，但是对于WKWebView的请求拦截不了，所以它无法在WKWebView中利用NSURLCache实现缓存。

#### 禁用NSURLCache（了解即可，基本不太可能用到）

如果我们的业务都是需要实时更新的话（如天气数据），则我们可以禁用缓存，很简单，只要将内存和磁盘空间设置为0即可。

```
NSURLCache *sharedCache = [[NSURLCache alloc] initWithMemoryCapacity:0
                                              diskCapacity:0
                                              diskPath:nil];
[NSURLCache setSharedURLCache:sharedCache];
```

#### 缓存策略

如果我们想要对`GET请求`进行数据缓存的话，直接使用`cachePolicy`。在NSURLConnection加载系统中，缓存被设计为request对象的一个属性，由NSURLRequest对象的cachePolicy属性指定。而在NSURLSession加载系统中，缓存被设计为NSURLSessionConfiguration对象的一个属性，该属性所指定的策略被该session的所有request所共享。

```
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];

// 设置缓存策略，只要设置了缓存策略，系统会自动利用NSURLCache进行数据缓存
request.cachePolicy = NSURLRequestReturnCacheDataElseLoad;

```

虽然上述我们指定了缓存策略，但是我们也可以在后台返回数据之后指定如何进行缓存（即当数据返回将要缓存时会调用后面的方法）。NSURLConnection 创建的request，其代理方法 `-connection:willCacheResponse`可以在缓存之前截断和编辑由 NSURLConnection 创建的 NSURLCacheResponse。同理，NSURLSession 中也有对应的方法 `URLSession:dataTask:willCacheResponse:completionHandler:`。

```
- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse {
    NSMutableDictionary *mutableUserInfo = [[cachedResponse userInfo] mutableCopy];
    NSMutableData *mutableData = [[cachedResponse data] mutableCopy];
    NSURLCacheStoragePolicy storagePolicy = NSURLCacheStorageAllowedInMemoryOnly;
 
    // ... 
    // 此处可以编辑返回的 cachedResponse
 
    return [[NSCachedURLResponse alloc] initWithResponse:[cachedResponse response]
                                                    data:mutableData
                                                userInfo:mutableUserInfo
                                           storagePolicy:storagePolicy];
}
 
 // 如果不想要缓存，则直接返回nil即可
- (NSCachedURLResponse *)connection:(NSURLConnection *)connection
                  willCacheResponse:(NSCachedURLResponse *)cachedResponse {
    return nil;
}
```

上述代码，我们也看到了另一个存储策略属性 storagePolicy，可以查看它有3个值。



iOS 对 NSURLRequest 默认的缓存策略如下：（实际上能使用的只有4种:1\2\5\6）

1. NSURLRequestUseProtocolCachePolicy 

	默认的缓存策略，它使用当前URL的协议中预置的缓存策略，无论这个协议是http，还是说自己定义的协议。对于常见的http协议来说，这个策略根据请求的头来执行缓存策略。服务器可以在返回的响应头重加入Expire策略或者Cache-Control策略来告诉客户端应该执行的缓存行为，同时配合#Last-Modified等头来控制刷新的时机。具体有关缓存的头信息下面会简单介绍。

2. NSURLRequestReloadIgnoringLocalCacheData 

   忽略缓存，重新请求，它根本不会缓存数据。
 
3. NSURLRequestReloadIgnoringLocalAndRemoteCacheData // 未实现
4. NSURLRequestReloadIgnoringCacheData = NSURLRequestReloadIgnoringLocalCacheData // 忽略缓存，重新请求

5. NSURLRequestReturnCacheDataElseLoad 

 	该策略它会一直尝试读取缓存数据，直到无法读取到缓存数据的时候，才会去请求网络。这样它有一个重大缺陷会让它根本无法被使用，因为它根本没有对缓存的刷新时机进行控制（比如过期了也不管，直接返回缓存的过期数据），如果你要去使用它，则需要额外的进行缓存过期控制。
 	
 	
6. NSURLRequestReturnCacheDataDontLoad // 有缓存就用缓存，没有缓存就不发请求，当做请求出错处理（用于离线模式）

7. NSURLRequestReloadRevalidatingCacheData // 未实现

如果我们自己去实现（像上述给出的完整缓存代码），则不用关心介绍的缓存策略，缓存策略完全是自己的实现代码决定。

如果我们想要在上述几个默认选项基础上，对缓存进行精确的控制或者修改，可以实现NSURLProtocol子类。NSURLProtocol 是个非常强大的类，不过它是个抽象类，所以需要实现它的子类。它可以拦截所有的请求，包括UIWebView中发起的请求（不过新的WKWebView中请求无法被拦截，WKWebView的请求都是在单独的进程中，所以不走NSURLProtocol），你可以修改request，或者response。所以这么看起来，NSURLProtocol 有点像中间人，它可以做很多事情。

### 网络协议中有关缓存的字段

#### Expires

Expires的值为服务端返回的到期时间，即下一次请求时，请求时间小于服务端返回的到期时间，直接使用缓存数据。
不过Expires 是HTTP 1.0的东西，现在默认浏览器均默认使用HTTP 1.1，所以它的作用基本忽略。
另一个问题是，到期时间是由服务端生成的，但是客户端时间可能跟服务端时间有误差，这就会导致缓存命中的误差。
所以HTTP 1.1 的版本，使用Cache-Control替代。


#### Cache-Control 头

在第一次请求到服务器资源的时候，服务器需要使用Cache-Control这个响应头来指定缓存策略，它的格式如下：Cache-Control:max-age=xxxx，这个头指指明缓存过期的时间
Cache-Control头具有如下选项（默认为你private）:

1. public: 任何路径的缓存者（本地缓存、代理服务器），可以无条件的缓存
2. private: 只针对单个用户或者实体（不同用户、窗口）缓存资源，比如客户端可以缓存
3. no-cache: response会被缓存到本地，但是请求时不使用本地缓存。
4. no-store: 指定不应该缓存
5. max-age: 指定过期时间
6. min-fresh:
7. max-stable:

#### Last-Modified/If-Modified-Since

Last-Modified 是由服务器返回响应头，标识资源的最后修改时间。

If-Modified-Since 则由客户端发送，标识客户端所记录的，资源的最后修改时间。服务器接收到带有该请求头的请求时，会使用该时间与资源的最后修改时间进行对比，如果发现资源未被修改过，则直接返回HTTP 304而不返回包体，告诉客户端直接使用本地的缓存。否则响应完整的消息内容。

#### Etag/If-None-Match

Etag 由服务器发送，告之当资源在服务器上的一个唯一标识符。

客户端请求时，如果发现资源过期(使用Cache-Control的max-age)，发现资源具有Etag声明，这时请求服务器时则带上If-None-Match头，服务器收到后则与资源的标识进行对比，决定返回200或者304。

以上几个头信息之间的关系及区别如下：

配置Last-Modified/ETag的情况下，浏览器再次访问统一URI的资源，还是会发送请求到服务器询问文件是否已经修改，如果没有，服务器会只发送一个304回给浏览器，告诉浏览器直接从自己本地的缓存取数据；如果修改过那就整个数据重新发给浏览器；

Cache-Control/Expires则不同，如果检测到本地的缓存还是有效的时间范围内，浏览器直接使用本地副本，不会发送任何请求。两者一起使用时，Cache-Control/Expires的优先级要高于Last-Modified/ETag。即当本地副本根据Cache-Control/Expires发现还在有效期内时，则不会再次发送请求去服务器询问修改时间（Last-Modified）或实体标识（Etag）了。

一般情况下，使用Cache-Control/Expires会配合Last-Modified/ETag一起使用，因为即使服务器设置缓存时间, 当用户点击“刷新”按钮时，浏览器会忽略缓存继续向服务器发送请求，这时Last-Modified/ETag将能够很好利用304，从而减少响应开销。

你可能会觉得使用Last-Modified已经足以让浏览器知道本地的缓存副本是否足够新，为什么还需要Etag（实体标识）呢？HTTP1.1中Etag的出现主要是为了解决几个Last-Modified比较难解决的问题：

1. Last-Modified标注的最后修改只能精确到秒级，如果某些文件在1秒钟以内，被修改多次的话，它将不能准确标注文件的新鲜度
2. 如果某些文件会被定期生成，当有时内容并没有任何变化，但Last-Modified却改变了，导致文件没法使用缓存
3. 有可能存在服务器没有准确获取文件修改时间，或者与代理服务器时间不一致等情形

Etag是服务器自动生成或者由开发者生成的对应资源在服务器端的唯一标识符，能够更加准确的控制缓存。Last-Modified与ETag是可以一起使用的，服务器会优先验证ETag，一致的情况下，才会继续比对Last-Modified，最后才决定是否返回304。

说明点：

1. Etag是服务器自动生成或者由开发者生成的对应资源在服务器端的唯一标识符，能够更加准确的控制缓存，但是需要注意的是分布式系统里多台机器间文件的last-modified必须保持一致，以免负载均衡到不同机器导致比对失败，Yahoo建议分布式系统尽量关闭掉Etag(每台机器生成的etag都会不一样，因为除了 last-modified、inode 也很难保持一致)。

2. Last-Modified/If-Modified-Since要配合Cache-Control使用，Etag/If-None-Match也要配合Cache-Control使用。


#### AFNetworking网络库缓存

仔细阅读AFNetworking源码，会发现它由一个图片缓存 `AFImageCache`，实际上它继承NSCache，只是内存缓存而已。网络接口缓存（NSURLSession版本），涉及到接口缓存代码如下：

```
// 有通过方法让外部赋值 dataTaskWillCacheResponse 这个block，可以指定如何缓存
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
 willCacheResponse:(NSCachedURLResponse *)proposedResponse
 completionHandler:(void (^)(NSCachedURLResponse *cachedResponse))completionHandler
{
    NSCachedURLResponse *cachedResponse = proposedResponse;

    if (self.dataTaskWillCacheResponse) {
        cachedResponse = self.dataTaskWillCacheResponse(session, dataTask, proposedResponse);
    }

    if (completionHandler) {
        completionHandler(cachedResponse);
    }
}

```

```
// 设置缓存策略，外部可以直接通过属性来设置
- (void)setCachePolicy:(NSURLRequestCachePolicy)cachePolicy {
    [self willChangeValueForKey:NSStringFromSelector(@selector(cachePolicy))];
    _cachePolicy = cachePolicy;
    [self didChangeValueForKey:NSStringFromSelector(@selector(cachePolicy))];
}
```

所以 AFNetworking 并没有提供现成的缓存方案，如果你基于AFNetworking做二次封装的话，可以顺便也把网络接口缓存也封装下，比如我们上述给定的完整方案。


## NSCache

此次也顺便说下另一个内存缓存，和上述的NSURLCache虽都是缓存，但除此之外没有其他共同之处了。NSCache 是一种内存缓存。 SDWebImage 就使用它作为内存缓存。

NSCache 它存储的是 key-value 对，这和 NSMutableDictionary 有点类似。所以有人会说，为啥我们不使用 NSMutableDictionary 来声明一个变量做缓存呢？（其实使用 NSCache 类会更好）NSCache 和 NSMutableDictionary 的区别如下：

1. NSCache 类结合了各种自动删除策略，以确保不会占用过的系统内存。如果其他应用需要内存时，系统自动执行这些策略。当调用这些策略时，会从缓存中删除一些对象，以最大限度减少内存的使用。如果使用 NSMutableDictionary ，则需要自己编写挂钩，在系统发出“低内存”（low memory）通知时手工删除缓存。而NSCache 会自己删除，由于其是 Foundation 框架的一部分，所以与开发者相比，它能在更深的层面上插入挂钩。

2. NSCache 中的 key 只会被强引用，不需要实现 NSCopying 协议。而 NSMutableDictionary 中的 key 使用的是 copy，所以自定义的类如果要做为 key ，则它必须要实现 copyWithZone 协议才可以。NSCache 对象不 copy 键的原因在于：很多时候，键都是不支持 copy 操作的对象来充当的。因此，在键不支持 copy 的情况下，NSCache 使用起来比 NSMutableDictionary 更方便。

3.  NSCache  是线程安全的，我们可以在不同线程中添加、删除和查询缓存中的对象，而不需要锁定缓存区域。

     这些特性对于 NSCache 类来说都是必须的，因为在需要释放内存时，缓存必须要异步的在幕后决定自动修改自身。


### NSCache 使用介绍

#### 基本属性

NSCache 提供了几个属性来限制缓存大小：

1.  @property  NSUInteger  countLimit;

     countLimit 限定了缓存最多维护的对象的个数。默认值为0，表示不限制数量。不过需要注意的是，这也不是一个严格的限制。如果缓存中的数量超过这个数量，缓存中的一个对象可能会被立即丢弃、或者稍后、也可能永远不会，具体依赖于缓存的实现细节。

2.   @property  NSUInteger  totalCostLimit;

      totalCostLimit 用来限定缓存能维持的最大内存。默认值为0， 表示没有限制。当我们添加一个对象到缓存中时，我们可以为其制定一个消耗（cost），如对象的字节大小。如果添加这个对象到缓存导致缓存总的消耗超过 totalCostLimit 的值，则缓存会自动丢弃一些对象，直到总消耗低于 totalCostLimit 指定的值。不过被丢弃的对象顺序是无法保证的。需注意的是：totalCostLimit 也不是一个严格限制。



#### 存取 key - value

NSCache 提供了一组方法来存取 key - value （如下用法同 NSMutableDictionary，只是  key 不需要像 NSMutableDictionary 中的 key 那样实现 NSCopying 协议，它只会强引用），如下：

1. -(id)objectForKey:(id)key

2. -(void)setObject:(id)obj  forKey:(id)key

3. -(void)removeObjectForKey:(id)key

4. -(void)removeAllObjects

我们在存储对象时，也可以为对象指定一个消耗值，如下：`-(void)setObject:(id)obj  forKey:(id)key   cost:(NSUInteger)num` 这个消耗值用于计算缓存中所有对象的一个消耗总和。当内存受限或者总消耗超过了限定的最大总消耗，则缓存应该开启一个丢弃过程以移除一些对象。不过，这个过程不能保证被丢弃对象的顺序。
     
通常情况下，这个消耗值是对象的字节大小。如果这些信息不是现成的，则我们不应该去计算它，因为这样会增加使用缓存的成本。如果我们没有可用的值传递，则直接传递0，或者使用 setObject:forKey: 方法，这个方法是不需要穿入一个消耗值。


NSCache 类还提供了一个属性，用来标识缓存是否自动舍弃那些内存已经被丢弃的对象，声明如下：

@property  BOOL  evictsObjectsWithDiscardedContent

如果设置为 YES，则表示在对象的内存被丢弃时舍弃对象。默认值为 YES。

#### 代理属性

NSCache 对象还有一个代理属性，声明如下：

@property (assign)  id <NSCacheDelegate>  delegate;

实现 NSCacheDelegate 代理的对象会在对象即将从缓存中移除时执行一些特定操作。因为代理对象可以实现如下方法：

- (void)cache:(NSCache *)cache  willEvictObject:(id)obj

特别需要注意的是，该代理方法中我们不可以修改 cache 对象，能做的也就是打印结果，查看，用于调试。













